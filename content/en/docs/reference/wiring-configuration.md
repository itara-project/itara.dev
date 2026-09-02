---
title: Wiring Configuration
description: The YAML format that declares topology — nodes, connections, and everything a connection carries.
weight: 4
---
 
The wiring configuration is the YAML file that describes the topology of an
Itara deployment: which nodes exist, and how they're connected to one
another (or to the outside world).
 
This document describes the file format itself — the schema, the required
and optional fields, and what makes a document valid.
 
## Top-level structure
 
```yaml
nodes:
  - id: "gatewayNode"
    component: "gateway"
  - id: "calculatorNode"
    component: "calculator"
 
connections:
  - id: "gateway-to-calculator"
    from: "gatewayNode"
    to: "calculatorNode"
    transport:
      id: http
      params:
        host: "${CALC_HOST:-localhost}"
        port: "${CALC_PORT:-8081}"
    serializer:
      id: json
```
 
| Field | Type | Required | Description |
|---|---|---|---|
| `nodes` | list of [Node](#nodes) | No (defaults to empty) | Every node in the topology. |
| `connections` | list of [Connection](#connections) | No (defaults to empty) | Every connection between nodes. |
 
An empty or comment-only file is valid and parses to an empty
configuration (no nodes, no connections).
 
## Environment variable substitution
 
Before the YAML is parsed, the raw file content goes through a textual
substitution pass. Any occurrence of:
 
- `${VAR_NAME}` — replaced with the value of environment variable `VAR_NAME`.
- `${VAR_NAME:-default}` — replaced with the value of `VAR_NAME` if set,
  otherwise the literal `default`.
is substituted in-place, before the YAML parser ever sees the file. This
means a substituted value is always plain text at parse time — e.g. an env
var substituted into a `port:` field arrives as a numeric string and the
YAML parser coerces it the same way a literal `8081` would be.
 
A variable with no default that isn't set in the environment is left as
the literal placeholder text (`${VAR_NAME}`) in the document.
 
Substitution applies anywhere in the file — it's a plain text pass over the
whole document, not scoped to particular fields.
 
## YAML merge keys and anchors
 
Standard YAML anchors/aliases and merge keys (`<<`) are supported and
resolved before the document is mapped onto the config model — so shared
blocks (e.g. a common `transport.params` map used by several connections)
can be factored out with a YAML anchor and merged in with `<<`.
 
## Nodes
 
A node is either a **component node** or a **virtual node**. The two share
an `id` and are distinguished by the `kind` field.
 
```yaml
nodes:
  - id: "orderServiceNode"
    component: "order-service"        # component node (kind omitted → defaults to "component")
 
  - id: "inventoryNode"
    kind: component                   # component node (kind explicit)
    component: "inventory"
 
  - id: "orderPlacedChannel"
    kind: virtual
    contract: "order-events/order-placed"
    address: "demo.events.order-placed"
```
 
### Common fields
 
| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | **Yes** | This node's own identifier. Must be unique, non-blank and match the [id character set](#id-character-set). Used elsewhere in the file (`connections[].from` / `connections[].to`) to refer to this node. |
| `kind` | string: `component` \| `virtual` | No — defaults to `component` | Discriminates which node type this is. Case-insensitive (`Component`, `VIRTUAL`, etc. are all accepted and normalized). A config without a `kind` field is a config full of component nodes. |
 
### Component nodes (`kind: component`)
 
A component node is a deployable component instance — something with an
activator and a contract implementation.
 
| Field | Type | Required | Description |
|---|---|---|---|
| `component` | string | **Yes** | The component id. Must match the `artifact.id` of a `kind = "component"` [metadata file](/docs/reference/metadata-format/) discoverable by the agent. |
 
### Virtual nodes (`kind: virtual`)
 
A virtual node represents a communication channel with no component
implementation of its own — a named point that decouples producers from
consumers via a broker (e.g. a Kafka topic). It has no activator and no
agent-managed lifecycle.
 
| Field | Type | Required | Description |
|---|---|---|---|
| `contract` | string | **Yes** | The event contract this channel carries, in `<events-artifact-id>/<contract-id>` form, e.g. `"order-events/order-placed"`. |
| `address` | string | **Yes** | The broker-specific channel address, e.g. a Kafka topic name. Opaque to the wiring config itself — meaning is entirely up to the transport/broker in use. |
 
### Id character set
 
Both node ids and connection ids are restricted to:
 
```
[A-Za-z0-9._-]+
```
 
i.e. letters, digits, `.`, `_`, and `-`. This set is deliberately
conservative — it's Kafka's own topic-name character set, and it stays safe
unencoded in HTTP header values and URL path segments, both of which a
transport implementation is free to use to propagate an id.
 
## Connections
 
A connection declares how one node calls another — or how an external
caller reaches into the topology.
 
```yaml
connections:
  - id: "gateway-to-calculator"
    from: "gatewayNode"
    to: "calculatorNode"
    transport:
      id: http
      handleTimeout: true
      params:
        host: "${CALC_HOST:-localhost}"
        port: "${CALC_PORT:-8081}"
    serializer:
      id: json
      params:
        schemaRegistryUrl: "${SCHEMA_REGISTRY_URL:-http://localhost:8081}"
    failureSemantics:
      id: built-in
      timeout: 2s
      handleTimeout: true
      absoluteTimeout: 10s
      maxRetry: 3
      params:
        waitDuration: 500ms
    authentication:
      id: shared-secret
      params:
        secret: "${GATEWAY_SECRET}"
    authorization:
      id: rule-table
      params:
        allow: "shout"
        deny: "whisper"
```
 
### Top-level connection fields
 
| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | **Yes** | This connection's own identifier — distinct from `from`/`to`, which identify *nodes*. Must be unique across the entire wiring configuration, not just per-node. Same [character set](#id-character-set) as node ids. |
| `from` | string | No | The calling node's id. Absent, `null`, or blank means the caller is external to the Itara topology — this connection defines an inbound entry point for `to`. |
| `to` | string | **Yes** | The called node's id. |
| `transport` | [TransportEntry](#transport-block) | **Yes** | The transport this connection uses. |
| `serializer` | [SerializerEntry](#serializer-block) | Conditionally — see below | The serializer this connection uses. Required for every connection except direct (colocated) ones. |
| `failureSemantics` | [FailureSemanticsEntry](#failuresemantics-block) | No | Retry, timeout, and circuit-breaking policy for this connection. Absent means the noop implementation is used — no retries, no enforced timeout, no circuit breaking. |
| `authentication` | [AuthenticationEntry](#authentication-block) | No | Authentication mechanism for this connection. Absent means the noop implementation is used. |
| `authorization` | [AuthorizationEntry](#authorization-block) | No | Authorization mechanism for this connection. Absent means the noop implementation is used. |
 
A connection with `from` absent/blank is an **external** connection — it
represents traffic entering the topology from outside (e.g. a public HTTP
endpoint), rather than a call between two nodes both managed by Itara.
 
A connection whose `transport.id` is (case-insensitively) `"direct"` is a
**direct** connection — an in-process, colocated call. Direct connections
never cross a process boundary, so nothing on them is ever serialized, and
they don't require a `serializer` block. A direct connection cannot be
external — an in-process call has no meaning without a caller also managed
by Itara.
 
### Transport block
 
```yaml
transport:
  id: http
  handleTimeout: true
  params:
    host: "${CALC_HOST:-localhost}"
    port: "8081"
```
 
| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | **Yes** | The transport type identifier. Must match the `artifact.id` of a `kind = "transport"` [metadata file](/docs/reference/metadata-format/) discoverable by the agent — except for the special value `direct`, which selects the built-in in-process transport and requires no corresponding artifact. |
| `handleTimeout` | boolean | No — defaults to `false` | Whether the transport should enforce the per-attempt timeout natively for this connection. Only meaningful if the transport's own metadata declares the matching capability; the actual timeout value lives in `failureSemantics.timeout`, not here. |
| `params` | map of string → scalar | No — defaults to `{}` | Transport-specific connection parameters, passed through to the transport implementation as-is. The wiring config has no schema for this map and no knowledge of what any particular transport expects. Values must be scalars (string, number, or boolean); nested maps or lists are not valid here. |
 
### Serializer block
 
```yaml
serializer:
  id: json
  params:
    schemaRegistryUrl: "${SCHEMA_REGISTRY_URL:-http://localhost:8081}"
```
 
| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | **Yes** (see [connection-level rule](#top-level-connection-fields)) | The serializer id. Must match the `artifact.id` of a `kind = "serializer"` [metadata file](/docs/reference/metadata-format/) discoverable by the agent. |
| `params` | map of string → scalar | No — defaults to `{}` | Serializer-specific parameters, passed through as-is. Same shape and same "no schema" property as `transport.params`. |
 
The block (and its `id` in particular) is required on every connection
except direct ones — there is no serializer choice that's safe to assume
silently for a connection that crosses a process boundary.
 
### failureSemantics block
 
```yaml
failureSemantics:
  id: built-in
  timeout: 2s
  handleTimeout: true
  absoluteTimeout: 10s
  maxRetry: 3
  params:
    waitDuration: 500ms
```
 
| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | No — defaults to `noop` | The failure-semantics implementation id. Must match the `artifact.id` of a `kind = "failure-semantics"` [metadata file](/docs/reference/metadata-format/), unless left at the `noop` default. |
| `timeout` | duration string, e.g. `"2s"`, `"500ms"` | No | Per-attempt timeout. |
| `handleTimeout` | boolean | No — defaults to `false` | Whether the failure-semantics implementation should enforce the per-attempt timeout by externally interrupting the transport thread. Only meaningful if the implementation's own metadata declares support for it. |
| `absoluteTimeout` | duration string, e.g. `"10s"` | No | Hard ceiling on total execution time across all attempts, including retries. |
| `maxRetry` | integer | No | Maximum number of retries. Total attempts made = `maxRetry + 1`. |
| `params` | map of string → scalar | No — defaults to `{}` | Implementation-specific parameters (e.g. backoff configuration), passed through as-is. |
 
The block as a whole is optional; omitting it entirely means the noop
implementation is used — no retries, no timeout enforcement, no circuit
breaking.
 
### authentication block
 
```yaml
authentication:
  id: shared-secret
  params:
    secret: "${GATEWAY_SECRET}"
```
 
| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | No — defaults to `noop` | The authentication mechanism id. Must match the `artifact.id` of a `kind = "authentication"` [metadata file](/docs/reference/metadata-format/), unless left at the `noop` default. If the block is present, `id` may not be blank — omit the block entirely to get the noop default instead of supplying an empty id. |
| `params` | map of string → scalar | No — defaults to `{}` | Implementation-specific parameters, passed through as-is. |
 
### authorization block
 
```yaml
authorization:
  id: rule-table
  params:
    allow: "shout"
    deny: "whisper"
```
 
| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | No — defaults to `noop` | The authorization mechanism id. Must match the `artifact.id` of a `kind = "authorization"` [metadata file](/docs/reference/metadata-format/), unless left at the `noop` default. Same non-blank rule as `authentication` when the block is present. |
| `params` | map of string → scalar | No — defaults to `{}` | Implementation-specific parameters, passed through as-is. |
 
## Full example
 
```yaml
nodes:
  - id: "order-service-node"
    component: "order-service"
  - id: "pricing-service-node"
    component: "pricing-service"
  - id: "orderPlacedChannel"
    kind: virtual
    contract: "order-events/order-placed"
    address: "demo.events.order-placed"
 
connections:
  # In-process call — order-service calls pricing-service directly.
  - id: "order-to-pricing"
    from: "order-service-node"
    to: "pricing-service-node"
    transport:
      id: direct
 
  # Inbound HTTP entry point — no 'from', so the caller is external.
  - id: "gateway-to-order"
    to: "order-service-node"
    transport:
      id: http
      params:
        host: "${ORDER_HOST:-localhost}"
        port: "${ORDER_PORT:-8080}"
    serializer:
      id: json
    authentication:
      id: shared-secret
      params:
        secret: "${GATEWAY_SECRET}"
 
  # order-service publishes onto a virtual (broker-backed) channel.
  - id: "order-to-order-placed-channel"
    from: "order-service-node"
    to: "orderPlacedChannel"
    transport:
      id: kafka
      params:
        bootstrapServers: "${KAFKA_BOOTSTRAP:-localhost:9092}"
    serializer:
      id: protobuf
```
