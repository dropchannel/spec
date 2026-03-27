# ADR-XXXX: Double-Dash Separator Between Protocol and Waterway Name

**Status:** Accepted  
**Deciders:** Paul Czajka  
**Date:** [to be set at commit time]  
**Governs:** All repositories in the `dropchannel/` org

## Context

DropChannel Waterway names have historically used a single-dash prefix convention to indicate the protocol handling a Waterway. For example:

- `riverway-my-chat`
- `tideway-updates`

This convention is ambiguous: a single dash is also valid within a protocol name (e.g., the potential future `meta-ripple` protocol) and within a user-selected Waterway name. Given a name like `meta-ripple-heartbeat`, it is not possible to determine by inspection alone where the protocol identifier ends and the user-selected name begins.

The introduction of meta-protocols (`meta-ripple`, `meta-spill`) makes this ambiguity concrete rather than theoretical.

## Decision

Waterway names will use a double-dash (`--`) as the separator between the protocol identifier and the user-selected Waterway name:

```
<protocol-identifier>--<user-selected-name>
```

### Examples

| Protocol | User Name | Waterway Name |
|---|---|---|
| `riverway` | `my-chat` | `riverway--my-chat` |
| `tideway` | `diagnostics` | `tideway--telemetry` |
| `meta-ripple` | `heartbeat` | `meta-ripple--heartbeat` |
| `meta-spill` | `telemetry` | `meta-spill--telemetry` |

### Parsing rule

The protocol identifier is the substring preceding the first occurrence of `--`. The user-selected name is the substring following it. Both the protocol identifier and the user-selected name may contain single dashes; neither may contain `--`.

## Consequences

- Waterway name parsing is unambiguous for all current and anticipated protocol identifiers, including multi-word meta-protocol names.
- The `PROTOCOL_REGISTRY` dispatch logic must be updated to split on `--` rather than the first `-`.
- User-selected Waterway names may not contain `--`; this should be validated at Waterway creation time.
- There are no existing deployments or Waterway names in the wild; this is a clean break with no migration required.
- All protocol specs (`riverway`, `tideway`, `meta-ripple`, `meta-spill`) should reference this convention when documenting Waterway naming.
