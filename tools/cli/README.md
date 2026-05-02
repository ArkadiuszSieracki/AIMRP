# AIMRP CLI

Version: 0.1  
Sprint: 6

The AIMRP CLI (`aimrp`) is the primary developer tool for managing peer nodes, inspecting the network, and launching reasoning sessions.

## General Rules

- Commands use kebab-case for both commands and flags: `peer start`, `--peer-id`, `--session-id`
- Boolean flags use `--flag` / `--no-flag` pattern
- Every command supports `--help` for usage info
- `--output json` switches any command to machine-readable JSON output
- Exit code `0` on success, non-zero on any error
- All network commands (peer status, network inspect, network peers, session start/status) automatically send `X-AIMRP-Version: 0.1` header; peers reject requests without it
- All commands that talk to the network require `--address` (peer or orchestrator endpoint) unless a default is set in config

## Commands

### `aimrp peer`

Manage local peer node lifecycle.

#### `aimrp peer start`

Start a peer node using the specified config file.

```
aimrp peer start [--config <path>] [--log-level <level>]
```

| Flag | Default | Description |
|---|---|---|
| `--config` | `./peer.yaml` | Path to peer config file |
| `--log-level` | `info` | Log verbosity: `debug`, `info`, `warn`, `error` |

Exit codes:
- `0` — peer running and DHT joined
- `1` — config error
- `2` — model backend unavailable
- `3` — DHT join failed

#### `aimrp peer stop`

Send shutdown signal to a running peer.

```
aimrp peer stop [--address <host:port>]
```

#### `aimrp peer status`

Print current peer status: identity, roles, model backend reachability, DHT status.

```
aimrp peer status [--address <host:port>] [--output json]
```

Example output (plain):
```
peer_id:   a3f9c2...
roles:     planner, reasoner, critic
model:     llama3.2:3b @ http://localhost:11434/v1  OK
dht:       joined, manifest TTL 3547s remaining
uptime:    00:14:32
```

#### `aimrp peer keygen`

Generate a new Ed25519 keypair and write it to the specified path.

```
aimrp peer keygen [--out <path>]
```

| Flag | Default | Description |
|---|---|---|
| `--out` | `./peer-key.pem` | Output path for keypair file |

Prints `peer_id` of the generated key to stdout.

---

### `aimrp network`

Inspect the AIMRP network.

#### `aimrp network peers`

List peers visible in the DHT, optionally filtered by role.

```
aimrp network peers [--bootstrap <host:port>] [--role <role>] [--output json]
```

| Flag | Default | Description |
|---|---|---|
| `--bootstrap` | from config | DHT bootstrap node address |
| `--role` | (all) | Filter peers by role: `planner`, `reasoner`, `critic`, `retriever` |

Example output (plain):
```
PEER_ID        ROLES               ADDRESS                   TTL
a3f9c2...      planner,reasoner    localhost:8080            3547s
8d12e1...      critic              peer2.example.com:8080    1820s
```

#### `aimrp network inspect`

Fetch and display capabilities of a specific peer.

```
aimrp network inspect --address <host:port> [--output json]
```

Calls `GET /capabilities` on the target peer and pretty-prints the response.

---

### `aimrp session`

Launch and manage reasoning sessions.

#### `aimrp session start`

Start a new reasoning session through an orchestrator.

```
aimrp session start --goal "<text>" [--orchestrator <host:port>] [--output json]
```

| Flag | Default | Description |
|---|---|---|
| `--goal` | (required) | Natural language goal for the session |
| `--orchestrator` | from config | Orchestrator endpoint |

Prints `session_id` and result summary to stdout on completion.

#### `aimrp session status`

Query status of an existing session.

```
aimrp session status --session-id <id> [--orchestrator <host:port>] [--output json]
```

---

### `aimrp config`

Validate and inspect config files.

#### `aimrp config validate`

Validate a peer or orchestrator config file against the schema.

```
aimrp config validate --file <path> [--type peer|orchestrator]
```

Exit code `0` if valid, `1` if invalid (prints field-level errors).

#### `aimrp config init`

Generate a starter config file interactively or with flags.

```
aimrp config init [--type peer|orchestrator] [--out <path>] [--backend <openai-compatible|anthropic>]
```

---

## Configuration

The CLI reads a global config from `~/.aimrp/config.yaml` (or `$AIMRP_CONFIG`):

```yaml
defaults:
  orchestrator: "localhost:9090"
  bootstrap: "localhost:4000"
  log_level: info
  output: plain          # plain | json
```

Override any default with an explicit flag on each command.

## Implementation Notes

- CLI is a .NET console application using `System.CommandLine`
- Use `Spectre.Console` for rich plain-text output (tables, status spinners)
- Use `Spectre.Console.Testing` for CLI unit tests
- JSON output mode: serialize the same domain object as `--output plain` uses; no separate code path
- All network calls use `IHttpClientFactory`; timeout default 10s; configurable via `--timeout <seconds>`
- Private key operations run in-process; key material never written to stdout or logs
