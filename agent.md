# DropChannel Agent — Functional Specification

*spec/ repository — `agent.md`*
*Status: Draft*

---

## Overview

The **Agent** is the host-level supervisor process for a DropChannel deployment on a
physical machine. One Agent runs per physical host. It owns the node's configuration,
manages the lifecycle of one or more **Workers**, aggregates telemetry and logging, and
represents the host as a single participant in the DropChannel observability layer.

This specification is implementation-agnostic. It defines what the Agent and its Workers
do, not how they are built in any specific runtime.

---

## Concepts and Terminology

| Term | Definition |
|------|------------|
| **Agent** | The long-running host-level supervisor process. One per physical host. |
| **Worker** | A subprocess spawned by the Agent to perform forwarding for a single channel. One Worker per configured channel direction. |
| **Node** | The protocol-level concept: a single forwarding hop in a Winch (or other protocol) pipeline. A Worker instantiates one Node. |
| **Agent Config** | The single configuration file owned by the Agent for the entire host. |
| **Backend** | A named ChannelProvider configuration. Defined once in Agent Config, referenced by Workers. |

The term **Node** retains its protocol-level meaning throughout the DropChannel spec
ecosystem. Agent and Worker are runtime concepts that exist below the protocol layer.

---

## Agent Responsibilities

The Agent is responsible for:

1. **Configuration ownership** — reading and holding the single host-level config file;
   distributing relevant subsections to Workers at spawn time.
2. **Worker lifecycle management** — spawning, monitoring, restarting, and stopping
   Workers.
3. **Log aggregation** — capturing structured output from Workers and writing a unified
   log for the host.
4. **Telemetry participation** — maintaining the host's single slot in the `telemetry-`
   channel, representing the aggregate state of all Workers.
5. **Signal handling** — responding to OS signals for graceful shutdown and config reload.

---

## Configuration

### File

One configuration file per physical host. Format: TOML. Path is provided to the Agent
at startup (e.g. as a command-line argument or environment variable). The Agent reads
the config on startup and on receipt of `SIGHUP` (reload).

### Structure

The config has five top-level sections: `agent`, `log`, `telemetry`, `backends`, and `workers`.

#### `[agent]`

General host-level settings.

```toml
[agent]
node_id = "a unique stable identifier for this host"
telemetry_refresh_interval_seconds = 60
```

- `node_id` — stable identifier for this host. Used as the Agent's identity in the
  `telemetry-` channel slot name and in log output. Must be unique across all hosts
  sharing any common storage backend.
- `telemetry_refresh_interval_seconds` — how often the Agent rewrites its telemetry slot
  even when no lifecycle event has occurred. Default: 60. Minimum: 10.

#### `[log]`

Controls local log output from the Agent. Workers emit structured JSON to stdout;
the Agent captures this and writes to the configured log destinations.

```toml
[log]
format = "json"          # "json" | "text"
stderr = true            # emit to stderr (default: true)
file = "/var/log/dropchannel/agent.log"  # emit to file (optional)
level = "info"           # "debug" | "info" | "warn" | "error"
```

- `format` — applies to all log destinations equally.
- `stderr` and `file` are independent; both may be active simultaneously.
- If neither `stderr` nor `file` is set to an active destination, the Agent MUST emit
  a startup warning and default to `stderr = true`.

#### `[telemetry]`

Configures the telemetry provider — the storage backend the Agent writes its telemetry
slot to. This is independent of `[backends]`: it is not named, not referenced by workers,
and not a general-purpose backend. It is typically a shared, dedicated storage location
serving all nodes in a deployment, allowing a single monitoring tool to read telemetry
from all participants without access to per-node payload backends.

```toml
[telemetry]
channel_id = "telemetry-dropchannel-prod"
type = "gcs"
bucket = "dropchannel-telemetry"
credentials = "/path/to/sa.json"
```

- `channel_id` — the `telemetry-` channel this Agent writes its state blob to.
- `type`, `bucket`, `credentials` (and other provider-specific fields) — ChannelProvider
  configuration for the telemetry backend, using the same field conventions as `[backends]`
  entries.

#### `[backends]`

Named ChannelProvider configurations. Each backend is defined once here and referenced
by name in worker configs. Workers do not hold their own backend credentials.

```toml
[backends.my-gcs]
type = "gcs"
bucket = "dropchannel-prod"
credentials = "/path/to/service-account.json"

[backends.local-relay]
type = "local"
path = "/tmp/dropchannel"
```

Backend keys (e.g. `my-gcs`) are arbitrary identifiers, unique within the config file.

#### `[workers]`

One subsection per Worker. Each Worker instantiates one protocol Node for one channel
direction.

```toml
[workers.foo-recv]
channel_id = "winch-documents"
recv_backend = "my-gcs"
send_backend = "local-relay"
poll_interval_seconds = 5

[workers.foo-send]
channel_id = "winch-documents"
recv_backend = "local-relay"
send_backend = "my-gcs"
poll_interval_seconds = 5
```

- Worker keys (e.g. `foo-recv`) are arbitrary identifiers, unique within the config
  file. They serve as the Worker's identity in logs and telemetry.
- `channel_id` — the full channel identifier including protocol prefix (e.g.
  `winch-documents`). The protocol is dispatched from this prefix per the protocol
  registry.
- `recv_backend` and `send_backend` reference backend keys defined in `[backends]`.
  They may be the same backend (single-provider hop) or different backends
  (cross-provider hop).

---

## Worker Lifecycle

### Spawning

Workers are spawned as subprocesses by the Agent. The Agent delivers each Worker's
configuration via `stdin` at spawn time as a single JSON object. Workers do not read
the Agent Config file directly.

The JSON delivered to a Worker's stdin contains:

- The worker's own config subsection (channel_id, poll_interval, etc.)
- The resolved backend configs for its `recv_backend` and `send_backend`
- The worker's assigned identity key (for log and telemetry attribution)

Workers MUST read and parse their config from stdin before beginning any protocol
activity. Workers MUST NOT require environment variables or additional files for
their core configuration.

### Steady State

Once running, a Worker:

- Executes the protocol state machine for its assigned channel and protocol.
- Emits structured JSON log lines to its own stdout at appropriate points (see
  [Worker Telemetry Emission](#worker-telemetry-emission)).
- Runs indefinitely until terminated by the Agent or until it exits due to an error.

### Restart Policy

The Agent restarts any Worker that exits for any reason (crash, non-zero exit code,
or unexpected clean exit).

#### Backoff

Restarts use exponential backoff to avoid hammering a backend that is causing
repeated failures:

| Restart attempt | Delay before next spawn |
|-----------------|------------------------|
| 1 | 1 second |
| 2 | 2 seconds |
| 3 | 4 seconds |
| 4 | 8 seconds |
| 5 | 16 seconds |
| 6+ | 30 seconds (cap) |

The backoff counter resets after the Worker has run continuously for 5 minutes without
exiting.

#### Crash Threshold and Degraded State

A crash counts toward the threshold only if the Worker exits within 60 seconds of
being spawned. If the Worker runs for more than 60 seconds before exiting, that exit
does not increment the crash counter (it is treated as an operational exit, not a
rapid crash).

If a Worker accumulates 5 threshold-qualifying crashes, it enters **degraded state**:

- The Agent continues restarting the Worker (with backoff).
- The Agent emits a `worker.degraded` telemetry event (distinct from `worker.restarted`).
- The degraded state is reflected in the telemetry slot blob for the host.

The crash counter and degraded state reset after the Worker runs continuously for
5 minutes without exiting.

### Shutdown

On receipt of `SIGTERM` or `SIGINT`, the Agent:

1. Stops spawning new Workers.
2. Sends `SIGTERM` to all running Workers and waits for graceful exit (configurable
   timeout, default 10 seconds).
3. Sends `SIGKILL` to any Workers that have not exited within the timeout.
4. Writes a final `agent.stopped` log event and telemetry update.
5. Exits cleanly.

On receipt of `SIGHUP`, the Agent reloads its config file and reconciles the running
Worker set: spawning Workers for any newly added worker configs, and stopping Workers
for any removed worker configs. Running Workers whose config has not changed are not
restarted.

---

## Logging

### Architecture

Workers emit structured JSON log lines to their stdout. The Agent captures each
Worker's stdout stream and annotates each line with the Worker's identity before
writing to the configured log destinations. The Agent also emits its own events
(daemon lifecycle, worker lifecycle) into the same stream.

The result is a single unified log for the entire host, with every line attributed
to either the Agent or a named Worker.

### Log Event Shape

Every log event is a JSON object with at minimum:

```json
{
  "ts": "2026-03-17T14:23:00.000Z",
  "level": "info",
  "source": "agent" | "<worker-key>",
  "event": "<event-name>",
  ...
}
```

Additional fields are event-specific. Unknown fields MUST be preserved by the Agent
when relaying Worker events (no field stripping).

When `format = "text"`, the Agent renders each event as a human-readable line:

```
2026-03-17T14:23:00Z [info] [foo-recv] worker.started channel_id=winch-documents pid=12345
```

### Agent Lifecycle Events

| Event | When emitted |
|-------|-------------|
| `agent.started` | Agent process has initialized and is ready to spawn Workers |
| `agent.config_loaded` | Config file successfully read (on startup or SIGHUP) |
| `agent.config_error` | Config file could not be read or parsed |
| `agent.stopping` | Agent has received a shutdown signal |
| `agent.stopped` | All Workers have exited; Agent is about to exit |

### Worker Lifecycle Events (emitted by Agent)

| Event | When emitted |
|-------|-------------|
| `worker.spawned` | Agent has spawned the Worker subprocess |
| `worker.exited` | Worker subprocess exited; includes exit code |
| `worker.restarting` | Agent is scheduling a restart; includes backoff delay |
| `worker.degraded` | Worker has crossed the crash threshold |
| `worker.recovered` | Worker's crash counter has reset after sustained clean run |
| `worker.stopping` | Agent has sent SIGTERM to the Worker |
| `worker.killed` | Agent has sent SIGKILL to the Worker (timeout exceeded) |

### Worker Telemetry Emission (emitted by Worker)

Workers emit the following events to their own stdout (which the Agent captures):

| Event | When emitted |
|-------|-------------|
| `worker.started` | Worker has parsed config and is beginning protocol activity |
| `worker.poll` | Completion of a poll cycle (debug level; may be suppressed at info) |
| `worker.state_transition` | Protocol state machine changed state |
| `worker.storage_error` | A backend operation returned an error |
| `worker.storage_ok` | Recovery after a previous storage error |

Workers MUST NOT emit events about their own lifecycle (spawn, exit, restart) — those
are owned by the Agent.

---

## Telemetry Participation

### Slot Ownership

The Agent owns exactly one slot in the `telemetry-` channel. No Worker writes to the
telemetry channel directly. The slot name is derived from `node_id`.

### Write Triggers

The Agent rewrites its telemetry slot on two triggers:

1. **Event-driven:** any Worker lifecycle transition (spawned, exited, restarted,
   degraded, recovered) causes an immediate slot rewrite.
2. **Periodic:** the Agent rewrites the slot on a fixed interval
   (`telemetry_refresh_interval_seconds`) even if no lifecycle event has occurred.
   This allows external monitors to detect a dead Agent by observing a stale slot.

### Telemetry Blob Schema

The blob written to the `telemetry-` slot is a JSON object:

```json
{
  "schema_version": 1,
  "node_id": "<agent node_id>",
  "agent_version": "<implementation version string>",
  "agent_started_at": "<ISO8601>",
  "reported_at": "<ISO8601>",
  "workers": [
    {
      "worker_id": "<worker key from config>",
      "channel_id": "<full channel_id>",
      "state": "<state>",
      "pid": 12345,
      "started_at": "<ISO8601 | null>",
      "restart_count": 0,
      "last_restart_at": "<ISO8601 | null>",
      "degraded": false,
      "last_activity_at": "<ISO8601 | null>"
    }
  ]
}
```

#### Worker `state` values

| Value | Meaning |
|-------|---------|
| `starting` | Agent has spawned the Worker; Worker has not yet emitted `worker.started` |
| `running` | Worker is active and healthy |
| `crashed` | Worker exited unexpectedly; restart pending |
| `restarting` | Agent is in backoff delay before next spawn |
| `degraded` | Crash threshold crossed; still restarting but flagged |
| `stopping` | Agent has sent SIGTERM; awaiting exit |
| `stopped` | Worker was cleanly stopped by Agent (e.g. config reload removed it) |

#### `last_activity_at`

Updated whenever the Agent receives any log event from the Worker. Allows external
monitors to detect a hung Worker (alive but silent) as distinct from a crashed Worker.

---

## Security Considerations

- The Agent Config file contains backend credentials. It SHOULD be readable only by
  the user account running the Agent (e.g. mode `0600`).
- Worker config is delivered via stdin at spawn time. It MUST NOT be written to disk
  as a temporary file.
- Workers MUST NOT hold backend credentials beyond what is required for their own
  two-backend operation. Workers receive only their own backend configs, not the full
  backend map.
- The Agent MUST NOT log backend credentials at any log level.
- Telemetry blobs are plaintext and visible to anyone with access to the telemetry
  storage backend. They MUST NOT contain credentials, channel payload content, or
  the shared secret. They MAY contain `node_id`, `worker_id`, `channel_id`,
  timestamps, and state values.

---

## Relationship to Other Spec Documents

| Document | Relationship |
|----------|-------------|
| `channel-provider.md` | Defines the ChannelProvider interface that Workers use for all backend operations |
| `protocol-registry.md` | Defines how Workers dispatch protocol behavior from `channel_id` prefix |
| `heartbeat.md` | Workers participate in the heartbeat protocol independently of the Agent; the Agent does not mediate heartbeat activity |
| `telemetry.md` | Defines the `telemetry-` channel semantics; this document extends it with the Agent blob schema |
| `observability.md` | Provides the overview of the two-layer observability model (heartbeat + telemetry) |
