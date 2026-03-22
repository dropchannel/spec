# Heartbeat

**Status:** Specified (introduced in dropchannel-py v0.6)
**Prefix:** `heartbeat-`
**Layer:** Protocol liveness — intra-pipeline observability
**Related:** `observability.md`, `telemetry.md`, `channel-provider.md`, `protocol-registry.md`

---

## Purpose

The heartbeat protocol provides continuous, per-hop liveness signaling within a running
DropChannel pipeline. It operates independently of primary payload traffic and introduces
no coupling to primary Waterway semantics.

Heartbeat addresses two observability gaps present in Tideway pipelines without it:

1. A stalled pipeline is indistinguishable from a pipeline with an offline participant.
2. In a multi-hop chain, there is no mechanism to identify which specific hop is
   responsible for a stall.

Both gaps are resolved entirely within the existing `DockProvider` interface via a
meta Waterway layer. No new storage operations are introduced. No out-of-band channels are
required.

**Heartbeat is not:**

- A payload transport. Heartbeat files carry no application data.
- A telemetry channel. Heartbeat operates within the pipeline's own storage positions;
  telemetry uses a dedicated side-channel. See `observability.md` for the distinction.
- An external monitoring dependency. Heartbeat state is readable by participants in the
  pipeline without any external tooling.

---

## Meta Waterways

Each Raft and client gains two additional storage namespaces alongside its existing
primary Waterways:

```
upper-side:  [ primary upper-side Waterway ]  [ meta upper-side Waterway ]
lower-side:  [ primary lower-side Waterway ]  [ meta lower-side Waterway ]
```

Meta Waterways are logically separate namespaces within the same `DockProvider` position.
They hold only heartbeat files. There is no coupling, sequencing dependency, or shared
state between the primary and meta layers. Primary Waterway behavior is entirely unaffected
by any meta Waterway state.

Meta Waterway storage representations per provider are documented in `channel-provider.md`.

---

## Participant identity

Each Raft and client instance has a persistent UUID, generated on first startup and
reloaded on all subsequent startups. This UUID is the stable identity used in heartbeat
file naming and content.

UUID format: standard UUID4.

---

## Heartbeat file naming

```
heartbeat-raft-{UUID}      # written by a Raft
heartbeat-client-{UUID}    # written by a client (endpoint)
```

One file per participant, keyed by UUID. Files are written to meta Waterways using
overwrite semantics — each write replaces the previous file at that name.

---

## Heartbeat content format

### Raft heartbeat

```
{upstream_content}
Raft_{UUID}:{timestamp}:{alive_s}
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
| `upstream_content` | string | Raft only. The full contents of the most recent non-self heartbeat file found on the relevant meta Waterway, or the literal string `HEARTBEAT_NOT_FOUND`. |

`upstream_content` is prepended to the Raft's own line, making each Raft's heartbeat
file a cumulative chain of all upstream participant heartbeats it has observed.

---

## Raft heartbeat cycle

Each Raft runs a heartbeat cycle at interval `HEARTBEAT_INTERVAL` (HI), concurrently
with and independently of the primary Waterway state machine.

**Step 1 — Read Lower-side meta Waterway, write Upper-side meta Waterway:**

```
Find any file matching heartbeat-raft-* or heartbeat-client-*
on the lower-side meta Waterway where filename != heartbeat-raft-{UUID_self}

If found:   content = contents of that file
If not:     content = "HEARTBEAT_NOT_FOUND"

Write to upper-side meta Waterway:
  filename: heartbeat-raft-{UUID_self}
  content:  content + "\n" + "Raft_" + UUID_self + ":" + timestamp + ":" + alive_s
```

**Step 2 — Read Upper-side meta Waterway, write Lower-side meta Waterway:**

```
Find any file matching heartbeat-raft-* or heartbeat-client-*
on the upper-side meta Waterway where filename != heartbeat-raft-{UUID_self}

If found:   content = contents of that file
If not:     content = "HEARTBEAT_NOT_FOUND"

Write to lower-side meta Waterway:
  filename: heartbeat-raft-{UUID_self}
  content:  content + "\n" + "Raft_" + UUID_self + ":" + timestamp + ":" + alive_s
```

**Step 3 — Sleep HI. Repeat from Step 1.**

If more than one non-self heartbeat file is found on a given meta Waterway (either side), the Raft
selects the most recently written and logs a warning. This condition should not arise
in a correctly configured pipeline.

---

## Client heartbeat

Clients write only their own `heartbeat-client-{UUID}` file. Clients do not read,
relay, or propagate any other heartbeat files — that is the Raft's responsibility.

Client heartbeats are **event-driven at status transitions**, then HI-driven until
the next transition:

- On each status transition: write a heartbeat immediately and reset the HI timer.
- Every HI interval thereafter while status is unchanged: write a heartbeat with the
  same status, updated `timestamp`, `alive_s`, and `status_s`.

Each client write goes to **both** Upper-side and Lower-side meta Waterways.

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
heartbeat files from both meta Waterways:

- A Raft deletes all files matching `heartbeat-raft-*` from both its Upper-side and
  Lower-side meta Waterways.
- A client deletes all files matching `heartbeat-client-*` from both its Upper-side and
  Lower-side meta Waterways.

Each participant purges only its own file type. Files written by adjacent participants
(e.g., a `heartbeat-client-*` file on a Raft's Lower-side meta Waterway) are left intact.

The purge ensures that stale heartbeat files from a previous session do not persist
across restarts. Recovery from the purge gap is bounded at 1×HI.

---

## Reading the heartbeat chain

Because Rafts relay upstream content forward, a participant at any position in the
pipeline can read the full liveness state of all upstream participants from a single
meta Waterway file.

**Full chain healthy** — a Raft's Lower-side heartbeat file contains entries from all
upstream participants in order, oldest upstream first:

```
Client_AAA:1710000000:120:ready:45
Raft_BBB:1710000001:119:
Raft_CCC:1710000002:118:
```

**Broken chain** — a `HEARTBEAT_NOT_FOUND` prefix indicates the Raft immediately
upstream of the reading participant has not produced a heartbeat. Entries after
`HEARTBEAT_NOT_FOUND` are from the reading Raft itself:

```
HEARTBEAT_NOT_FOUND
Raft_CCC:1710000002:118:
```

This identifies the stall location: the hop between the Raft writing `HEARTBEAT_NOT_FOUND`
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

A participant not implementing the heartbeat protocol (e.g., a v0.5 Raft in a v0.6
pipeline) will not write heartbeat files. It will appear as `HEARTBEAT_NOT_FOUND` at
its adjacent Rafts — indistinguishable from an offline participant. The pipeline
continues to function; that hop is simply invisible to heartbeat diagnostics.

In a mixed pipeline, `HEARTBEAT_NOT_FOUND` cannot be used to distinguish an offline
participant from a non-participating one. Operators should note this limitation when
upgrading incrementally.

---

## Protocol registry entry

The following entry belongs in `protocol-registry.md`:

| Prefix        | Protocol | Description |
|---------------|----------|-------------|
| `heartbeat-`  | Meta Waterway | Per-hop liveness chain. Rafts relay upstream heartbeat content forward; clients write status signals. Operates on meta Waterways alongside primary payload Waterways. Plaintext. |
