# ADR 0001 — Node flavors, the native node runtime, and repo structure

- Status: Proposed
- Date: 2026-07-09
- Deciders: JGA (koinos), Pablo
- Supersedes/extends: `docs/backlog/operations/MULTI_NODE_REMOTE_INSTALL_AND_MANAGEMENT_PLAN.md`

## Context

Koinos One today runs and manages one node type — the Teleno monolith — locally
and, via the existing SSH-backed fleet layer, on remote hosts. Two forward-looking
goals push beyond that:

1. The **multiservice node** (the official Koinos microservice stack: chain,
   mempool, block_store, p2p, jsonrpc, …) must be runnable **standalone from a CLI**
   on macOS today and Windows later, not only inside Koinos One.
2. Koinos One must be able to run and manage **more than Teleno**: the multiservice
   node, other compatible nodes, and nodes on **external servers** (install, start,
   upgrade, monitor, recover) — not merely connect to a remote RPC endpoint.

The existing remote plan already provides a rich foundation: fleet inventory,
node roles (observer/producer/standby), an SSH command-plan executor with
confirmation and sanitized receipts, runtime supervisors (Docker / systemd /
launchd), and a safety model. What it does not yet model is the node **type**
itself — everything is implicitly `teleno_node`.

Separately, building the microservice binaries natively on macOS ARM64 currently
depends on a knodel-local orchestrator (`build-native-mac.sh`) plus post-extraction
patch scripts. Those per-dependency compile fixes belong in the shared build config
(`koinos-cmake`), not in a downstream script — see the "Build layer" decision below.

## Decision

### 1. Model a node as three orthogonal axes

A managed node is the tuple **(flavor, location, supervisor)** with a common
lifecycle interface (install, start, stop, upgrade, rollback, health, logs,
backup/restore):

- **Flavor** — *what* node: `teleno-monolith`, `multiservice`, or other compatible
  implementations. This is the **new** axis; today it is implicit.
- **Location** — *where* it runs: `local` (direct process exec) or `remote`
  (SSH command-plan). Already implemented.
- **Supervisor** — *how* the process is kept alive: `launchd`, `systemd`, `docker`,
  or `foreground`. Already implemented.

The critical property: local and remote are the **same generated command-plan**
executed over a different transport (direct exec vs SSH). Adding the multiservice
node is therefore adding a new **flavor**, plugged into the executor that already
runs Teleno — not a separate subsystem.

The multiservice flavor differs from Teleno only in shape: Teleno is
"one binary + config + one supervisor unit"; multiservice is "N binaries +
GarageMQ + ordered start + N supervisor units + inter-service health". Same
lifecycle contract, more moving parts.

### 2. Extract a flavor-oriented native node runtime

Create a portable, headless **node runtime package** that knows, per flavor, how to
lay out, launch (in dependency order), configure (incl. GarageMQ for the
multiservice AMQP bus), supervise, and health-check a node on a host — with **no
GUI and no fleet/SSH logic**. It must be:

- **Portable**: droppable on any host, driven by a declarative command-plan.
- **Supervisor-agnostic**: emits launchd/systemd/docker unit definitions or runs
  foreground.
- **Cross-platform by design**: platform-specific only in binary resolution and
  process spawning; macOS now, Windows later without touching the axes.
- **Embeddable and CLI-able**: consumed both by a thin standalone CLI and by
  Koinos One's command-plan executor as the multiservice flavor adapter.

This is where today's `build-native-mac.sh` orchestration logic moves and is
rewritten as reusable modules. Because Koinos One is Electron (TS/Node), the runtime
SHOULD be a TS/Node library so it embeds directly; a bash orchestrator would force
fragile shell-out. (The per-dependency *compile* patches do not live here — they live
in `koinos-cmake`, see below.)

### 3. Build layer stays in koinos-cmake

Every per-dependency macOS/Windows compile fix (fizzy, zlib, abseil SSE, rocksdb
PORTABLE, gmp path, koinos_exception `_GNU_SOURCE`, …) is encoded as an
`if(APPLE …)` / platform conditional in `koinos-cmake`'s `Hunter/config.cmake`
(the in-progress koinos/koinos-cmake#18), NOT as post-extraction runtime patches in
a downstream script. End state: a plain per-repo `cmake` builds each microservice
on macOS with no wrapper. This dissolves most of the build orchestrator; what remains
is the runtime package of decision 2.

### 4. Repo structure

| Repo | Role | Axis |
|---|---|---|
| `koinos/koinos-cmake` | Build config: per-dependency mac/win compile patches in Hunter config | build |
| `koinos/koinos-node-runtime` (new) | Per-flavor portable runtime: ordered launch, GarageMQ config, supervisor unit templates, health probes. Consumed by the standalone CLI and by Koinos One | flavor + supervisor |
| `koinos/koinos-one` | Fleet, inventory, SSH executor, command-plan generation, safety model, GUI; adds the multiservice flavor adapter over the runtime | location + orchestration + UI |
| `koinos/koinos` | Docker umbrella (effectively the `docker` supervisor for Linux clusters) | — |

Microservice binaries are distributed via each service repo's GitHub Releases; the
runtime resolves or fetches them per flavor/version.

## Consequences

Positive:

- Adding a node type = adding a flavor value; no duplication across local/remote or
  CLI/GUI.
- `koinos-node-runtime up --flavor multiservice` gives developers a native stack
  without Docker or the GUI.
- Koinos One's node picker becomes `flavor × location`, reusing the existing SSH
  executor, safety model, and receipts.
- Windows support later is a new `supervisor` value + binary resolution; the axes are
  unchanged.
- Build reproducibility improves: `cmake` alone builds on macOS once koinos-cmake#18
  lands.

Costs / risks:

- A new repo (`koinos-node-runtime`) to own and version.
- Rewriting `build-native-mac.sh` orchestration as TS/Node modules.
- Koinos One's fleet model must promote "flavor" to a first-class, persisted field
  (migration of existing inventory records).
- The multiservice flavor's ordered-start + inter-service health is genuinely more
  complex than Teleno's single process; the runtime must model dependencies and
  readiness, not just spawn.

## Alternatives considered

- **Put the orchestrator in `koinos/koinos`.** Rejected: that repo is the Docker
  umbrella (compose-based cluster launch), a different concern from native
  build+run of individual flavors.
- **Keep orchestration as a macOS bash script Koinos One shells out to.** Rejected:
  not embeddable in Electron, not cross-platform, duplicates logic between CLI and
  GUI.
- **Model remote nodes as a thin "connect to RPC URL" provider only.** Rejected:
  the goal is full remote lifecycle (install/start/upgrade/manage), which the
  existing SSH command-plan layer already provides; a read-only connector is a
  strict subset (a `location: remote` node with lifecycle disabled).

## Open questions

- Runtime language confirmation (TS/Node assumed for Electron embeddability) and its
  packaging/distribution (npm package vs bundled).
- Binary distribution: per-service releases vs a signed multiservice bundle.
- Windows supervisor mapping (Windows Service vs a foreground supervisor shim).
- Whether "other compatible nodes" need a flavor SDK/plugin contract or a fixed enum
  initially.
