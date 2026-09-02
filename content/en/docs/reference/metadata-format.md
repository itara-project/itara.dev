---
title: Metadata Format
description: Identity and properties for anything that takes part in an Itara-managed topology — components, contracts, events, and plugins alike.
weight: 5
---
 
A `.itara` file is a small [TOML](https://toml.io/) document shipped
alongside every build artifact (a component, an API, a transport, a
serializer, an observer, etc). It declares what the artifact *is* — its
identity, the versions it was built against, and (depending on its kind) a
handful of kind-specific declarations that other tooling and the agent
itself use to wire things up safely.
 
Unlike the [wiring configuration](/docs/reference/wiring-configuration/),
which is one file describing an entire deployment topology, a `.itara`
file describes exactly one artifact.
 
This document describes the file format — every section, every field,
what's required, what defaults apply, and which artifact kinds each
section is meaningful for.

Wherever a field is described below as a version — `artifact.version`,
`api-version`, and every `version` inside a `{ id, version }` entry —
it's a semver string, not an arbitrary one; version-range fields (like
`[serializers] supported[].version`) are semver ranges checked against it.
 
## Forward compatibility
 
Unknown sections, and unknown fields within known sections, are always
silently ignored. This is deliberate: it lets an older agent load an
artifact built with a newer version of the metadata schema without
failing, simply ignoring whatever it doesn't understand. Don't rely on
unrecognized fields causing a parse error — they won't.
 
## Top-level structure
 
```toml
[artifact]
kind = "component"
id = "inventory"
version = "1.0.0"
api-version = "1.x"
 
[runtime]
language = "java"
compiler = "21"
 
[itara]
spec-version = "0.1"
core-version = "0.1+"
```
 
| Section | Required | Meaningful for | Description |
|---|---|---|---|
| `[artifact]` | **Yes** | every kind | Identity: what this artifact is, and its own version. |
| `[runtime]` | No | every kind | What language/compiler built this artifact. |
| `[itara]` | No | every kind | Which version of the Itara spec/core this artifact targets. |
| `[methods]` | No | `api` | Per-method properties of the contract — currently, which methods are non-idempotent. |
| `[implemented-event-contracts]` | No | `component` | Which event contracts this component implements. |
| `[contract]` | No | `api`, `events` | Properties of the contract as a whole — currently, its message format. |
| `[serializers]` | No | `api` | Which serializers this artifact was compiled with support for. |
| `[serializer]` | No | `serializer` | What this serializer implementation is and can do. |
| `[transport]` | No | `transport` | What this transport implementation is and can do. |
| `[failure-semantics]` | No | `failure-semantics` | What this failure-semantics implementation can do. |
| `[authentication]` | No | `authentication` | What kind of authentication mechanism this is. |
| `[authorization]` | No | `authorization` | What kind of authorization mechanism this is. |
| `[api-dependencies]` | No | `component` | Which API contracts this component calls, and at what version. |
 
Every section beyond `[artifact]` is optional at the parser level. The
"meaningful for" column describes intent, not an enforced restriction —
nothing about the format itself rejects a `[transport]` section on a
`component` artifact, for instance.
 
## `[artifact]` (required)
 
```toml
[artifact]
kind = "component"
id = "inventory"
version = "1.0.0"
api-version = "1.x"
```
 
| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `kind` | string | **Yes** | — | The artifact's kind: `component`, `api`, `events`, `transport`, `serializer`, `observer`, `failure-semantics`, `authentication`, `authorization`, `context-handler`. |
| `id` | string | **Yes** | — | Component id for `component`/`api`/`events` artifacts (e.g. `"inventory"`); SPI implementation name for `transport`/`serializer`/`observer`/`failure-semantics`/`authentication`/`authorization` artifacts (e.g. `"http"`). |
| `version` | string | No | `""` | This artifact's own version. |
| `api-version` | string | No | `""` | The API version this artifact exposes (for `api`/`component` producing an API) or was compiled against. |
 
`.itara` files are named without an embedded version — identity comes
purely from `(kind, id)`.
 
## `[runtime]` (optional)
 
```toml
[runtime]
language = "java"
compiler = "21"
```
 
| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `language` | string | No | `""` | Implementation language, e.g. `"java"`, `"rust"`. |
| `compiler` | string | No | `""` | Compiler/language version used to build the artifact, e.g. `"21"`, `"1.78+"`. |
 
## `[itara]` (optional)
 
```toml
[itara]
spec-version = "0.1"
core-version = "0.1+"
```
 
| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `spec-version` | string | No | `""` | Version of the Itara spec this artifact was built against. |
| `core-version` | string | No | `""` | Version of the Itara core this artifact was built against. |
 
## `[methods]` (optional — `api` artifacts)
 
```toml
[methods]
non-idempotent = ["divide", "transfer", "placeOrder"]
```
 
| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `non-idempotent` | array of strings | No | `[]` | Names of contract methods that are not safe to retry automatically. |
 
Absence of this section (or of a given method name within it) means the
method is treated as idempotent — the safe default for retry purposes.
 
## `[implemented-event-contracts]` (optional — `component` artifacts)
 
```toml
[implemented-event-contracts]
contracts = [
  { id = "order-events/order-placed",    version = "1.0.0" },
  { id = "order-events/order-cancelled", version = "1.0.0" },
]
```
 
| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `contracts` | array of `{ id, version }` | No | `[]` | Event contracts this component implements. |
 
Each entry:
 
| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | **Yes** | Full contract reference: `<events-artifact-id>/<contract-id>`, e.g. `"order-events/order-placed"`. |
| `version` | string | **Yes** | Version of the events artifact this implementation was written against. |
 
Present only on component artifacts that consume event contracts.
 
## `[contract]` (optional — `api` and `events` artifacts)
 
```toml
[contract]
message-format = "protobuf"
```
 
| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `message-format` | string | No | `""` | The message format the contract's method parameter/return types (for `api`) or event payload types (for `events`) are generated from, e.g. `"protobuf"`. An empty string is treated identically to the section being absent entirely — both mean the contract uses plain, hand-written types rather than a generated structural format. |
 
This is a structural property of the contract's own types — it's unrelated
to which serializer ids an artifact is compatible with (that's
`[serializers]`, below); message format and serializer choice vary
independently.
 
## `[serializers]` (optional — `api` artifacts)
 
```toml
[serializers]
supported = [
  { id = "json", version = "1.x" },
  { id = "protobuf", version = "1.x" },
]
```
 
| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `supported` | array of `{ id, version }` | No | `[]` | Serializers this API artifact was compiled with support for. |
 
Each entry:
 
| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | **Yes** | Matches a serializer's `artifact.id`, e.g. `"json"`, `"protobuf"`. |
| `version` | string | **Yes** | A semver range checked against that serializer's own `artifact.version`. |

Not every language needs this section populated. Java loads serializers as
runtime plugins, so an API artifact can work with a serializer it never
declared here. Rust has no equivalent runtime plugin mechanism — a Rust
API artifact must be compiled with support for whichever serializers it
will use, so `[serializers] supported` is how that compiled-in support
gets declared and checked.
 
## `[serializer]` (optional — `serializer` artifacts)
 
```toml
[serializer]
type = "protobuf"
 
[serializer.capabilities]
message-formats = ["protobuf"]
```
 
| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `type` | string | No | absent/`null` | The serialization category this implementation handles — e.g. `"json"`, `"protobuf"`. Distinct from `artifact.id`, which is the unique identifier of the specific implementation. |
| `capabilities` | [SerializerCapabilities](#serializercapabilities) | No | see below | See below. |
 
### SerializerCapabilities
 
| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `message-formats` | array of strings | No | `[]` | Structural message formats this serializer implementation can handle generically, beyond plain hand-written types — e.g. a protobuf serializer declares `["protobuf"]` here, meaning it can handle proto-generated types via reflection. |
 
Defaults to an empty list when the `[serializer.capabilities]` section is
absent — a serializer is assumed to handle plain types only, until it
explicitly declares a structural format it supports. This has no bearing
on error-payload handling, which every serializer supports unconditionally
regardless of its declared `message-formats`.
 
## `[transport]` (optional — `transport` artifacts)
 
```toml
[transport]
type = "http"
 
[transport.capabilities]
native-call-timeout = true
externally-interruptible = true
```
 
| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `type` | string | No | absent/`null` | The transport category / communication protocol, e.g. `"http"`, `"kafka"`, `"amqp"`. Two implementations sharing the same `type` are considered compatible caller/callee pairs. Distinct from `artifact.id`. |
| `capabilities` | [TransportCapabilities](#transportcapabilities) | No | see below | See below. |
 
### TransportCapabilities
 
| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `native-call-timeout` | boolean | No | `true` | Whether the transport can abort an in-flight call natively when the per-call timeout expires. If `false`, the failure-semantics layer is responsible for enforcing the timeout externally instead. |
| `externally-interruptible` | boolean | No | `true` | Whether it's safe to externally interrupt the thread blocked on this transport's send call — i.e. doing so won't leave the transport in a broken state. |
 
Both fields default to `true` when the `[transport.capabilities]` section
is absent entirely — a transport is assumed capable unless it explicitly
declares otherwise. This is the opposite default from
`SerializerCapabilities` and `FailureSemanticsCapabilities`, both of which
default to the unsupported/empty case.
 
## `[failure-semantics]` (optional — `failure-semantics` artifacts)
 
```toml
[failure-semantics]
 
[failure-semantics.capabilities]
supports-external-timeout = true
```
 
| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `capabilities` | [FailureSemanticsCapabilities](#failuresemanticscapabilities) | No | see below | See below. |
 
### FailureSemanticsCapabilities
 
| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `supports-external-timeout` | boolean | No | `false` | Whether this failure-semantics implementation can enforce the per-attempt timeout by externally interrupting the transport thread. Implementations must opt in explicitly. |
 
## `[authentication]` (optional — `authentication` artifacts)
 
```toml
[authentication]
type = "mtls"
```
 
| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `type` | string | No | absent/`null` | The authentication mechanism category, e.g. `"shared-secret"`, `"mtls"`, `"jwt"`. Distinct from `artifact.id`. |
 
## `[authorization]` (optional — `authorization` artifacts)
 
```toml
[authorization]
type = "rbac"
```
 
| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `type` | string | No | absent/`null` | The authorization mechanism category, e.g. `"rbac"`, `"rule-table"`, `"opa"`. Distinct from `artifact.id`. |
 
## `[api-dependencies]` (optional — `component` artifacts)
 
```toml
[api-dependencies]
calls = [
  { id = "calculator", version = "1.0.0" },
  { id = "inventory",  version = "2.1.0" },
]
```
 
| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `calls` | array of `{ id, version }` | No | `[]` | Synchronous API contracts this component calls. |
 
Each entry:
 
| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | **Yes** | Matches the `artifact.id` of the callee's `kind = "api"` metadata file. |
| `version` | string | **Yes** | The exact version this component was compiled against — used to check compatibility against the callee's declared `api-version`. |
 
Absence of the section (rather than an empty `calls` list) means the same
thing: the component declares no outbound API calls. This is valid, and
expected, for leaf components.
 
## Full example
 
A component that calls one API and implements two event contracts:
 
```toml
[artifact]
kind = "component"
id = "order-service"
version = "1.4.0"
api-version = "1.x"
 
[runtime]
language = "java"
compiler = "21"
 
[itara]
spec-version = "0.1"
core-version = "0.2"
 
[api-dependencies]
calls = [
  { id = "pricing", version = "2.0.1" },
]
 
[implemented-event-contracts]
contracts = [
  { id = "order-events/order-placed",    version = "1.0.0" },
  { id = "order-events/order-cancelled", version = "1.0.0" },
]
```

An API artifact declaring one non-idempotent method:

```toml
[artifact]
kind = "api"
id = "order-service"
version = "1.4.0"

[methods]
non-idempotent = ["placeOrder"]
```
 
An HTTP transport artifact:
 
```toml
[artifact]
kind = "transport"
id = "http"
version = "0.3.0"
 
[runtime]
language = "rust"
compiler = "1.78+"
 
[transport]
type = "http"
 
[transport.capabilities]
native-call-timeout = true
externally-interruptible = true
```
