---
title: itara-cli
description: The command-line tool for inspecting and validating an Itara wiring configuration.
weight: 1
---
 
`itara` is the command-line tool for working with an Itara
[wiring configuration](/docs/reference/wiring-configuration/) — inspecting
a topology at a glance, and validating it for logical mistakes before it's
deployed. It's implemented once, in Rust, and operates purely on the file
formats — the wiring config YAML and `.itara` metadata TOML files — so it
works identically regardless of which language (Java or Rust) actually
produced the artifacts being wired together; there's simply nothing
language-specific for it to know about.
 
```
itara <COMMAND> [OPTIONS]
```
 
Two subcommands are available:
 
| Command | Purpose |
|---|---|
| [`itara inspect`](#itara-inspect) | Print a human-readable summary of the topology described by a wiring config. |
| [`itara verify`](#itara-verify) | Validate the logical correctness of a wiring config, optionally cross-checked against a directory of `.itara` metadata files. |
 
Both commands take the path to a wiring config file as their first positional
argument, parse it with the same loader described in the
[wiring configuration](/docs/reference/wiring-configuration/) doc (including environment
variable substitution and YAML merge-key resolution), and exit non-zero if
the file can't be read or parsed at all — before any command-specific logic
runs.
 
## Installation / building
 
There's no published binary yet — build it from the `itara-cli` crate in
the [source repository](https://github.com/itara-project/itara):
 
```sh
cargo build --release -p itara-cli
# binary at target/release/itara
```
 
or install it onto your `PATH`:
 
```sh
cargo install --path itara-cli
```
 
## `itara inspect`
 
```
itara inspect [OPTIONS] <CONFIG>
```
 
Prints a readable summary of a wiring config's topology: every node, every
internal connection, the deployment groups the topology implies, and an
ASCII arrow graph of how calls flow through it. Intended for humans
skimming a config, not for machine consumption — there's no `--format json`
or similar; it's plain text.
 
### Arguments
 
| Argument | Required | Description |
|---|---|---|
| `<CONFIG>` | **Yes** | Path to the wiring config file. |
 
### Options
 
| Option | Description |
|---|---|
| `--no-graph` | Suppress the `Graph` section. |
| `--no-groups` | Suppress the `Deployment groups (derived)` section. |
| `--no-events` | Suppress the `Emits:` / `Listens to:` lines within deployment groups. Only meaningful when groups are shown at all. |
| `--node <id>` | Show only the given node and its direct connections, instead of the whole topology. Repeatable — pass it multiple times to focus on a small neighbourhood of several nodes at once. An id that doesn't exist in the config produces a warning on stderr but isn't fatal. |
 
### What each section shows
 
**Nodes** — every node in the (possibly `--node`-filtered) config, one per
line, with its kind-specific detail (`component: <id>` or
`virtual: <contract> @ <address>`). A node with at least one external
inbound connection (a connection with no `from`) is marked
`(external entry point)`.
 
**Connections** — every *internal* connection (i.e. connections with a
non-empty `from` — external inbound connections are omitted from this list
since they have no interesting `from → to` shape to show), as
`<id>: <from> → <to> [<transport-id>]`.
 
**Deployment groups (derived)** — component nodes joined by one or more
`direct` (in-process/colocated) connections are grouped together, since
they'll end up running in the same process. This is a connected-components
search over the subgraph of `direct` edges only — every other transport
type is treated as crossing a deployment boundary. Groups are labeled `A`,
`B`, `C`, …, `Z`, `AA`, `AB`, … in the order their first node is
encountered (which follows node declaration order in the file). Virtual
nodes never form or join a group — grouping only concerns deployable
components. For each node in each group, its non-direct inbound/outbound
connections are also listed:
 
- `Receives: external <transport> on <port>` — an external inbound connection.
- `Receives: <from> via <transport> on :<port>` — an inbound connection from another node.
- `Calls: <to> via <transport>` — an outbound connection to another (non-virtual) node.
- `Emits: <to> (<contract>)` — an outbound connection to a virtual node (suppressed by `--no-events`).
- `Listens to: <from> (<contract>)` — an inbound connection from a virtual node (suppressed by `--no-events`).

This is the practical answer to "what goes together, what runs separately,
what needs to start before what" — the basis for generating Docker Compose
services, Kubernetes manifests, or startup ordering, derivable from the
config alone rather than reconstructed by reading code. The full example
below, from Itara's own demo, shows a realistic case: three groups, one of
them multi-node, with both synchronous and event-driven connections mixed
together.
 
**Graph** — an attempt to render the topology as arrow chains, e.g.:
 
```
[external] --http:8080--> [gatewayNode] --http:8081--> [calculatorNode]
```
 
Each external entry point starts its own chain, which is walked forward
through outbound connections as far as it can go in a straight line. The
walk stops the moment it reaches a **branch point** (a node with more than
one outbound connection) or a **merge point** (a node with more than one
inbound connection) — at that point the chain is left as-is, and any
connections not swept up into a compressed chain are rendered as their own
individual line instead, annotated with `(id: <connection-id>)` so an
otherwise-ambiguous branch can still be traced back to a specific entry in
the config. The section is omitted entirely when the config has no
connections at all.
 
### Exit codes
 
| Code | Meaning |
|---|---|
| `0` | Success. |
| `1` | The config file could not be read or parsed, or (when `--node` is combined with a config that fails re-validation after filtering) filtering failed. |
 
`--node` referencing an id that doesn't exist in the config is a warning on
stderr, not a failure — inspection still proceeds with whatever's left.
 
### Full example
 
From Itara's own demo — an order-processing system with five components
and three event channels, mixing `direct`, `http`, and `kafka` connections
in one config:
 
```yaml
nodes:
  - id: "fulfilmentNode"
    component: "fulfilment"
  - id: "inventoryNode"
    component: "inventory"
  - id: "notificationNode"
    component: "notification"
  - id: "paymentNode"
    component: "payment"
  - id: "orderNode"
    component: "order"
  - id: "orderReservedChannel"
    kind: virtual
    contract: "order-events/order-reserved"
    address: "demo.events.order-reserved"
  - id: "orderFulfilledChannel"
    kind: virtual
    contract: "fulfilment-events/order-fulfilled"
    address: "demo.events.order-fulfilled"
  - id: "orderCancelledChannel"
    kind: virtual
    contract: "fulfilment-events/order-cancelled"
    address: "demo.events.order-cancelled"
 
connections:
  - id: "external-to-order"
    from:
    to: "orderNode"
    transport:
      id: http
      params:
        host: "localhost"
        port: 8081
    serializer:
      id: "json"
  - id: "external-to-inventory"
    from:
    to: "inventoryNode"
    transport:
      id: http
      params:
        host: "localhost"
        port: 8081
    serializer:
      id: "json"
  - id: "order-to-inventory"
    from: "orderNode"
    to: "inventoryNode"
    transport:
      id: direct
  - id: "order-to-fulfilment"
    from: "orderNode"
    to: "fulfilmentNode"
    transport:
      id: direct
  - id: "order-to-payment"
    from: "orderNode"
    to: "paymentNode"
    transport:
      id: http
      handleTimeout: true
      params:
        host: payment
        port: 8083
    serializer:
      id: "json"
    failureSemantics:
      id: built-in
      maxRetry: 3
      timeout: 5s
      params:
        waitDuration: 500ms
        retryRuntime: "true"
  - id: "order-to-orderReserved"
    from: "orderNode"
    to: orderReservedChannel
    transport:
      id: kafka
      params:
        bootstrapServers: "kafka:29092"
    serializer:
      id: "json"
  - id: "order-to-orderFulfilled"
    from: "orderNode"
    to: orderFulfilledChannel
    transport:
      id: kafka
      params:
        bootstrapServers: "kafka:29092"
    serializer:
      id: "json"
  - id: "order-to-orderCancelled"
    from: "orderNode"
    to: orderCancelledChannel
    transport:
      id: kafka
      params:
        bootstrapServers: "kafka:29092"
    serializer:
      id: "json"
  - id: "orderReserved-to-notification"
    from: "orderReservedChannel"
    to: "notificationNode"
    transport:
      id: kafka
      params:
        bootstrapServers: "kafka:29092"
        consumerGroup: "order-consumer-group"
    serializer:
      id: "json"
  - id: "orderFulfilled-to-notification"
    from: "orderFulfilledChannel"
    to: "notificationNode"
    transport:
      id: kafka
      params:
        bootstrapServers: "kafka:29092"
        consumerGroup: "order-consumer-group"
    serializer:
      id: "json"
  - id: "orderCancelled-to-notification"
    from: "orderCancelledChannel"
    to: "notificationNode"
    transport:
      id: kafka
      params:
        bootstrapServers: "kafka:29092"
        consumerGroup: "order-consumer-group"
    serializer:
      id: "json"
```
 
```
$ itara inspect demo/wiring-monolith.yaml
Itara topology — demo/wiring-monolith.yaml
 
Nodes:
  fulfilmentNode        component: fulfilment
  inventoryNode         component: inventory                     (external entry point)
  notificationNode      component: notification
  paymentNode           component: payment
  orderNode             component: order                         (external entry point)
  orderReservedChannel  virtual:   order-events/order-reserved @ demo.events.order-reserved
  orderFulfilledChannel virtual:   fulfilment-events/order-fulfilled @ demo.events.order-fulfilled
  orderCancelledChannel virtual:   fulfilment-events/order-cancelled @ demo.events.order-cancelled
 
Connections:
  order-to-inventory:             orderNode → inventoryNode [direct]
  order-to-fulfilment:            orderNode → fulfilmentNode [direct]
  order-to-payment:               orderNode → paymentNode [http]
  order-to-orderReserved:         orderNode → orderReservedChannel [kafka]
  order-to-orderFulfilled:        orderNode → orderFulfilledChannel [kafka]
  order-to-orderCancelled:        orderNode → orderCancelledChannel [kafka]
  orderReserved-to-notification:  orderReservedChannel → notificationNode [kafka]
  orderFulfilled-to-notification: orderFulfilledChannel → notificationNode [kafka]
  orderCancelled-to-notification: orderCancelledChannel → notificationNode [kafka]
 
Deployment groups (derived):
  Group A: fulfilmentNode, orderNode, inventoryNode
    fulfilmentNode (fulfilment)
    orderNode (order)
      Receives: external http on :8081
      Calls:    inventoryNode via direct
      Calls:    fulfilmentNode via direct
      Calls:    paymentNode via http
      Emits:       orderReservedChannel (order-events/order-reserved)
      Emits:       orderFulfilledChannel (fulfilment-events/order-fulfilled)
      Emits:       orderCancelledChannel (fulfilment-events/order-cancelled)
    inventoryNode (inventory)
      Receives: external http on :8081
 
  Group B: notificationNode
    notificationNode (notification)
      Listens to:  orderReservedChannel (order-events/order-reserved)
      Listens to:  orderFulfilledChannel (fulfilment-events/order-fulfilled)
      Listens to:  orderCancelledChannel (fulfilment-events/order-cancelled)
 
  Group C: paymentNode
    paymentNode (payment)
      Receives: orderNode via http on :8083
 
Graph:
  [external] --http:8081--> [orderNode]
  [external] --http:8081--> [inventoryNode]
  [orderNode] --direct--> [inventoryNode]  (id: order-to-inventory)
  [orderNode] --direct--> [fulfilmentNode]  (id: order-to-fulfilment)
  [orderNode] --http:8083--> [paymentNode]  (id: order-to-payment)
  [orderNode] --kafka--> [orderReservedChannel]  (id: order-to-orderReserved)
  [orderNode] --kafka--> [orderFulfilledChannel]  (id: order-to-orderFulfilled)
  [orderNode] --kafka--> [orderCancelledChannel]  (id: order-to-orderCancelled)
  [orderReservedChannel] --kafka--> [notificationNode]  (id: orderReserved-to-notification)
  [orderFulfilledChannel] --kafka--> [notificationNode]  (id: orderFulfilled-to-notification)
  [orderCancelledChannel] --kafka--> [notificationNode]  (id: orderCancelled-to-notification)
```
 
`fulfilmentNode` and `inventoryNode` both join `orderNode`'s group despite
`inventoryNode` also having its own external entry point — group
membership follows the `direct`-connection subgraph only, independent of
what else a node is reachable from. `paymentNode` and `notificationNode`
end up in their own single-node groups because every connection reaching
them crosses a transport boundary — `http` for one, `kafka` for the
other — neither of which is `direct`.
 
## `itara verify`
 
```
itara verify [OPTIONS] <CONFIG>
```
 
Runs a battery of logical checks against a wiring config and reports every
problem found, each as an `ERROR` or a `WARNING`. Errors indicate the
config is broken; warnings flag something suspicious that's still valid.
Meant to be run in CI or pre-deploy — the exit code tells you pass/fail, the
output tells you why. Introducing a topology layer creates an obligation: a
wiring config that can be misconfigured silently is not a step forward.
 
### Arguments
 
| Argument | Required | Description |
|---|---|---|
| `<CONFIG>` | **Yes** | Path to the wiring config file. |
 
### Options
 
| Option | Description |
|---|---|
| `--metadata-dir <path>` | Path to a directory of `.itara` metadata files (see the [metadata format](/docs/reference/metadata-format/) reference). Enables the checks that need to cross-reference metadata — known transport/authentication/authorization types, API version compatibility, timeout capability, transport interrupt safety, and serializer compatibility. Without it, those checks are skipped and a warning is emitted saying so. |
| `--skip <check>` | Skip a specific check by name. Repeatable. Mutually exclusive with `--only`. |
| `--only <check>` | Run *only* the specified check(s). Repeatable. Mutually exclusive with `--skip`. |
 
An unrecognized name passed to either `--skip` or `--only` is a hard error
(exit `1`) *before* the config file is even read, listing every valid check
name. Passing both `--skip` and `--only` in the same invocation is rejected
by argument parsing itself.
 
### Checks
 
| Check name | Severity | Requires `--metadata-dir` | What it flags |
|---|---|---|---|
| `duplicate-ids` | Error | No | The same node `id` declared more than once. |
| `connection-id-uniqueness` | Error | No | The same connection `id` declared more than once. |
| `self-connections` | Error | No | A connection whose `from` and `to` are the same node. |
| `direct-external-conflict` | Error | No | A connection using the `direct` transport with no `from` — a direct call is in-process and can't have an external caller. |
| `outbound-ambiguity` | Error | No | A node with outbound connections to two or more different node ids that all resolve to the *same* component — the agent would have no way to pick one at dispatch time. |
| `orphaned-nodes` | Error | No | A node declared in `nodes` that no connection (as either `from` or `to`) ever references. |
| `orphaned-connections` | Error | No | A connection whose `from` or `to` references a node id that isn't declared in `nodes`. |
| `virtual-no-producers` | Warning | No | A virtual node with no inbound connection — nothing will ever publish to it. |
| `virtual-no-consumers` | Warning | No | A virtual node with no outbound connection — nothing will ever consume from it. |
| `virtual-transport-mismatch` | Warning | No | A virtual node whose inbound/outbound connections don't all agree on the same transport type. |
| `unknown-transport` | Error | **Yes** | A connection's `transport.id` (other than the built-in `direct`) has no matching `kind = "transport"` metadata file. |
| `unknown-authentication` | Error | **Yes** | A connection's `authentication.id` (other than the built-in `noop`) has no matching `kind = "authentication"` metadata file. |
| `unknown-authorization` | Error | **Yes** | A connection's `authorization.id` (other than the built-in `noop`) has no matching `kind = "authorization"` metadata file. |
| `api-version-compatibility` | Error | **Yes** | For a connection between two component nodes: the caller's declared `[api-dependencies]` version for the callee doesn't satisfy the callee's declared `api-version` requirement (checked with semver). Also errors (rather than silently skipping) if either side's metadata, or the specific dependency declaration, can't be found — see [caveats](#api-version-compatibility-caveats) below. |
| `timeout-capability` | Error and Warning | **Yes** | A connection declares a `failureSemantics.timeout` or `absoluteTimeout` that its transport and/or failure-semantics implementation isn't capable of enforcing the way it's configured — see [details](#timeout-capability-details) below. |
| `transport-interrupt-safety` | Error | **Yes** | A connection configures external timeout enforcement (`failureSemantics.handleTimeout = true`) on a transport whose metadata declares `externally-interruptible = false`. |
| `serializer-compatibility` | Warning | **Yes** | A connection's serializer isn't confirmed compatible with the callee API's declared `[contract]` message format or `[serializers]` supported list — see [details](#serializer-compatibility-details) below. |
 
Without `--metadata-dir`, only the first ten checks run; the metadata-backed
checks are skipped entirely, and a single warning is added to the output:
*"no --metadata-dir provided — API version, known transport, timeout, and
transport interrupt checks are skipped"*.
 
#### `api-version-compatibility` caveats
 
This check only evaluates connections between two component nodes (it skips
external connections, and connections touching a virtual node on either
side). Where it does apply, every one of the following also produces its
own `ERROR` rather than silently skipping the connection: no metadata found
for the caller component; no metadata found for the callee component; the
caller declares no `[api-dependencies]` at all; the caller's
`[api-dependencies]` doesn't include an entry for the callee; the caller's
declared dependency version isn't valid semver; or the callee doesn't
declare an `api-version` (or declares an invalid semver range for it). In
other words: if you've opted into this check via `--metadata-dir`, an
incomplete metadata picture is treated as a problem to fix, not something
to shrug off.
 
#### `timeout-capability` details
 
This check only looks at non-external, non-direct connections that declare
a `failureSemantics` block with a `timeout` and/or `absoluteTimeout`. Within
that scope it can raise several distinct issues on the same connection:
 
- **Error** — `transport.handleTimeout = true` but the transport's metadata
  says `native-call-timeout = false` (or the transport has no metadata at
  all).
- **Error** — `failureSemantics.handleTimeout = true` but the
  failure-semantics implementation's metadata says
  `supports-external-timeout = false` (or has no metadata at all).
- **Error** — `failureSemantics.handleTimeout = true` but the transport's
  metadata says `externally-interruptible = false`.
- **Warning** — a `timeout` is declared but *neither*
  `transport.handleTimeout` nor `failureSemantics.handleTimeout` is set —
  the value will be threaded through but nothing will actually enforce it.
- **Warning** — *both* `transport.handleTimeout` and
  `failureSemantics.handleTimeout` are set — both will independently try to
  enforce the timeout, and which one wins is a race.
- **Error** — `absoluteTimeout` is declared but the failure-semantics
  implementation doesn't declare `supports-external-timeout = true` (or has
  no metadata at all) — an absolute timeout can only ever be enforced
  externally, so this is required whenever it's set, independent of
  `handleTimeout`.
#### `serializer-compatibility` details
 
Skips `direct` connections (nothing is ever serialized on them) and
connections whose callee doesn't resolve to a component. The check itself
only applies when the callee API's metadata declares *something* to check
against — a non-empty `[contract].message-format` or a non-empty
`[serializers].supported` list. An API declaring neither produces no issue
at all for this check (not even a warning): there's no restriction to be
incompatible with. Where the check does apply, a connection's configured
serializer is considered compatible if *either*:
 
- it appears (by id, with a version satisfying the declared semver range)
  in the API's `[serializers].supported` list, **or**
- the API's declared `[contract].message-format` appears in the
  serializer's own `[serializer.capabilities].message-formats`.

If neither holds — or if the serializer's own metadata can't be found at
all — the connection is flagged with a `WARNING` (never an `ERROR`): this
check reports "we can't confirm this is safe," not "this is broken," since
the agent itself doesn't enforce serializer/API compatibility at runtime
either.
 
### Exit codes
 
| Code | Meaning |
|---|---|
| `0` | The config parsed successfully and no `ERROR`-severity issues were found. Warnings do not affect the exit code. |
| `1` | The config could not be read/parsed, an unrecognized `--skip`/`--only` check name was given, or at least one `ERROR`-severity issue was found. |
 
### Output format
 
```
<✓|✗> itara verify — <path>
 
  <N> nodes, <M> connections
 
  <label>  <message>
  <label>  <message>
  ...
 
  <summary line>
```
 
`✓` is printed when there are no errors (warnings may still be present);
`✗` when there is at least one error. If the config fails to parse, only
the header, a single `ERROR` line for the parse failure, and a `1 error`
summary are printed — none of the individual checks run. If there are no
issues at all (not even warnings — i.e. `--metadata-dir` was supplied and
everything passed), the line `No issues found.` is printed instead of a
check list and summary.
 
### Examples
 
A clean run against the demo's monolith topology, with a metadata
directory supplied:
 
```
$ itara verify --metadata-dir demo/metafiles/ demo/wiring-monolith.yaml
✓ itara verify — demo/wiring-monolith.yaml
 
  8 nodes, 11 connections
 
  No issues found.
```
 
The same topology with two problems deliberately introduced — the
`order-to-fulfilment` connection removed (orphaning `fulfilmentNode`, since
nothing else references it) and `handleTimeout` dropped from
`order-to-payment`'s transport block while a `timeout` is still declared —
triggers `orphaned-nodes` and `timeout-capability` together, one an error
and one a warning:
 
```
$ itara verify --metadata-dir demo/metafiles/ demo/wiring-informed-with-error.yaml
✗ itara verify — demo/wiring-informed-with-error.yaml
 
  8 nodes, 10 connections
 
  ERROR  node 'fulfilmentNode' is declared but not referenced in any connection
  WARN   connection 'orderNode' → 'paymentNode': a timeout is declared but neither the transport nor the failure semantics implementation is configured to enforce it — the timeout value will be passed to the transport but nothing will act on it
 
  1 error, 1 warning
```
 
Narrowing to a single check, against the same demo config:
 
```
$ itara verify --metadata-dir demo/metafiles/ demo/wiring-monolith.yaml --only unknown-transport
✓ itara verify — demo/wiring-monolith.yaml
 
  8 nodes, 11 connections
 
  No issues found.
```
