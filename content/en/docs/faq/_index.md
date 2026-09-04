---
title: FAQ & Comparisons
description: How Itara relates to service meshes, ESBs, hexagonal architecture, and other things people reach for first.
weight: 5
---

### What is Itara?

Itara is a specification for topology-as-configuration, with reference implementations in Java and Rust. The core idea: which components exist, how they communicate, and where they run are configuration decisions — not code decisions. Component logic expresses what the system does. The wiring config expresses how the parts connect. These are different concerns and they live in different places.
 
In practice: you write a plain interface and an implementation. The agent reads the wiring config at startup, wires the components together — directly in-process, over HTTP, or over any other supported transport — and hands off to your application. Your code never changes when the topology changes. Only the config changes.
 
---
 
### Is this a framework? A runtime? A service mesh?
 
None of those exactly. It is closer to a specification with reference implementations. The Java and Rust agents are engines that read the spec and apply it at startup. Your application compiles and runs as a normal binary. There is no Itara API in your business logic, no annotations on your domain objects, no framework base classes to extend.
 
The closest analogy is a classloader for distributed systems topology — something that wires things together at startup and then gets out of the way.
 
---
 
### What is Itara not trying to do?
 
Itara is a thin slice — it concentrates one specific concern, topology, into an explicit and executable layer. It is not trying to replace anything in the existing stack.
 
**Container orchestrators** (Kubernetes, Nomad, ECS) own deployment, scaling, and scheduling. Itara declares which components connect and how; orchestrators decide where they run and how many instances exist. These are complementary concerns.
 
**Service meshes** (Istio, Linkerd, Envoy) operate on network traffic between deployment units, typically as sidecars. Itara operates at the component level, above the network, with explicit contracts between components. A service mesh and Itara can coexist — they solve different problems at different layers.
 
**Dapr** provides building blocks (pub/sub, state stores, service invocation) as a sidecar API. Itara declares and realizes the communication structure between components without a sidecar in the call path at runtime. See the Dapr entry below for a longer comparison.
 
**Deployment and CI/CD tooling** (Terraform, Helm, ArgoCD) own how software gets to where it runs. Itara's tooling will eventually generate inputs for these systems from the declared topology, but it does not replace them.
 
**Observability platforms** (Datadog, Jaeger, Grafana, Kibana) own how telemetry is stored, queried, and visualised. Itara fires structured observability events at every component boundary through a pluggable observer SPI — any observability stack implementing it can consume them. Where that data goes is entirely the operator's choice.
 
The goal is not to own the stack. It is to make one currently invisible layer — topology — explicit, verifiable, and executable, and let everything else continue doing what it already does well.
 
---
 
### Isn't this the same as CORBA, Java RMI, or other transparency-oriented systems?
 
No, and the distinction matters.
 
CORBA, Java RMI, DCOM, and EJB all attempted to make remote calls indistinguishable from local calls — to hide the network from the developer entirely. The premise was that the network doesn't exist, or at least that you shouldn't have to think about it. This failed for well-documented reasons: the network does exist, it has latency and failure modes that local calls don't, and hiding those differences produces systems that are hard to reason about and harder to debug.
 
Itara's premise is the opposite. The network exists and its presence is always visible — in the wiring config, in the traces, in the error contracts. What Itara removes from component code is not the awareness that a call might be remote, but the responsibility for encoding how that remote call works. Failure semantics, serialization, transport selection, context propagation — these are topology concerns declared in the wiring config, not infrastructure boilerplate written into the business logic.
 
The difference in plain terms: CORBA made topology invisible. Itara makes it explicit and concentrated. Topology is declared, validated by tooling before deployment, and visible to everyone — the opposite of hidden.
 
The fact that this keeps getting attempted — CORBA, RMI, EJB, actor frameworks — is itself a signal the underlying problem is real, not a reason to dismiss another attempt. What's different here isn't the ambition, it's the mechanism.
 
---
 
### Isn't this just hexagonal architecture, DDD, and dependency injection?
 
Hexagonal architecture and DDD answer how to organize the *internals* of a component — its ports, its domain model, its bounded context. Itara doesn't compete with either; a well-designed hexagonal component is exactly the kind of thing that plugs cleanly into Itara's contract model.
 
The closer comparison is **Spring Modulith**, which does let modules communicate with configurable transport (in-memory or Kafka via event externalization) independent of business logic. That's real, adjacent territory. Two differences matter: Modulith's communication config lives *inside* the monolith's own codebase, coupled to annotations on the module/event itself — it has no model of independently deployed, independently versioned services. And it governs modules within one JVM deployable; it can't express "this module might become its own process later" without someone picking transport code at that point. Itara's wiring config is external to every component's code and treats placement — colocated or distributed — as a topology decision, not a modularity pattern chosen once at build time.
 
Dependency injection frameworks (Spring, Guice) wire dependencies within one process; they have no concept of a dependency living in a different process at all, which is the actual problem Itara is aimed at.
 
---
 
### Isn't this just workflow orchestration, like the Arazzo Specification?
 
No — different layer entirely. Arazzo describes *sequences of calls to already-existing, already-deployed APIs* to accomplish something: call this endpoint, take its output, feed it into that one, in this order. It assumes the APIs already exist as separate, deployed services and describes the choreography of calling them.
 
Itara doesn't describe business workflows — it describes the *structure* of the system those workflows run on top of: which components exist, how they connect, whether they're even separate processes at all. Arazzo has no concept of placement, transport, or whether two things need to be separate services in the first place — it's closer to a very structured API-calling script than to a topology declaration.
 
---
 
### Isn't this just Spring Integration (or another Enterprise Integration Patterns library)?
 
Closer than hexagonal/DDD, and worth taking seriously. Spring Integration does let you swap an in-memory channel for JMS/AMQP/Kafka without touching endpoint logic — real overlap.
 
The difference: Spring Integration's channels are still part of your application's code and config. You write `IntegrationFlow` beans, your handler methods take `Message<T>`, the framework is present in the business layer, not just at a boundary outside it. It's also fundamentally single-application scoped — bridging to an independently deployed, independently versioned service is an adapter you configure by hand, not a topology decision validated across two separately built artifacts before deploy. And there's no cross-language story; it's Spring, full stop.
 
One line version: Spring Integration decouples channels *within* one application. Itara decouples topology *across* independently built, independently deployed, potentially different-language components — and validates the connection before either side ships.
 
---
 
### How is this different from Dapr?
 
The difference is where the abstraction sits.
 
Dapr is a sidecar: a separate process injected alongside your container. Every call your service makes to another service goes through a localhost network hop to the Dapr sidecar, which routes it to the target's sidecar, which delivers it. This is true even when both services are on the same host. The sidecar tax is always paid.
 
Itara works at the method call level. When two components are declared colocated in the wiring config, the proxy resolves at startup and the call is a direct in-process function call — no serialization, no network, no sidecar. When they are remote, the call goes over HTTP or whichever transport is configured. The code is identical in both cases. Dapr cannot offer zero-overhead colocation because the sidecar model structurally prevents it.
 
There is also a granularity difference. Dapr operates at the service level — the unit of deployment is a service, and the sidecar wires services together. Itara operates at the component level, which is deliberately finer-grained than a service. Multiple components can be colocated in one process or distributed across separate processes — all through config changes. Service boundaries are not fixed deployment artifacts in Itara; they are adjustable topology decisions.
 
The other difference: Dapr abstracts infrastructure (state stores, brokers, secrets). Itara abstracts topology — where components run and how they connect. These are different problems.
 
---
 
### How does Itara compare to Microsoft Aspire?
 
They operate at different layers and solve different problems.
 
Aspire is a development and deployment orchestration tool — it declares which services and infrastructure resources exist, how they start up, and how they get deployed. It significantly improves the experience of running and shipping multi-service applications, particularly for .NET teams.
 
Itara operates one layer deeper: the communication contracts between components. It declares what components call, what they return, how failures are handled, and validates that every connection is correct before deployment. Where Aspire answers "how do I run and deploy my services consistently," Itara answers "what are the communication contracts between my components, are they valid, and can topology change without touching code." The two tools address complementary concerns and can coexist.
 
---
 
### What does zero-overhead colocation actually mean?
 
When two components share the same process and type system — two JVM components wired as `direct`, or two Rust components in the same process — the proxy calls the implementation directly. Nothing happens that isn't strictly necessary for the call itself: no serialization, no network hop, no transport overhead of any kind.
 
That's not the same as saying nothing happens at all. Observability events still fire. Authentication and authorization still run, if configured for that connection. Component scope is still established. Whatever else the topology layer legitimately needs to do for a call to be correct and safe still happens — the guarantee is that none of it is transport-shaped work a colocated call never needed in the first place, not that the call is literally free.
 
The corollary: running two components colocated in Itara costs nothing *beyond what a monolith would already cost you* for the equivalent call. The boundary itself is free. This is what makes topology switching meaningful — moving from colocated to remote changes what shows up in your traces as network and serialization cost, without changing the topology-layer guarantees around the call.
 
---
 
### If two components are colocated, do they trust each other?
 
No — colocation is a placement decision, made to optimise communication overhead, not a trust decision. Every executing component has an agent-established scope defining exactly what it can reach, and that boundary is enforced identically whether a connection is colocated or crosses a process. A component cannot supply its own identity, and it cannot reach another component it has no declared connection to — regardless of whether they happen to share a process. This is what makes it safe to colocate independently built components for performance without also having to trust them with unrestricted access to each other.
 
Authentication and authorization are pluggable SPIs on top of this, opt-in per connection, and — same as the scope guarantee underneath them — behave identically whether the connection is colocated or remote.
 
---
 
### Topology is no longer in the component code — how do I handle network failures?
 
Topology is not hidden — it is explicitly declared in the wiring config, where it is visible, auditable, and validated before deployment. What is absent from component code is the responsibility for encoding how failures travel, not the awareness that failures can happen.
 
Failure semantics — retries, timeouts, circuit breaking — are connection-level configuration, not component-level code. They belong in the wiring config alongside the transport declaration. The component code never sees infrastructure boilerplate, but the failure contracts are explicit and declared.
 
The failure semantics SPI is pluggable. A single implementation owns the complete strategy for a connection — retry logic, timeout enforcement, circuit breaking — as a cohesive unit of behaviour declared in configuration. A built-in implementation ships with the platform covering the common cases. Teams with specific requirements can provide their own.
 
Failures surface as typed error contracts at the call site: `CHECKED` for declared component errors, `RUNTIME` for unexpected component failures, `TRANSPORT` for infrastructure failures, and `PERMISSION` for a caller that wasn't authenticated or authorized. All four reconstruct as the same exception type on the caller side — one thing to catch, regardless of which kind of failure it was.
 
---
 
### With topology out of the component code, doesn't that make debugging harder?
 
With topology being more visible than ever before, tracking down errors actually becomes easier.
 
At the code level, every failure that crosses a component boundary surfaces as a typed error contract, carrying the original exception class and message where applicable. Without reading a line of transport code, you know immediately whether the failure was in the business logic, the component implementation, the infrastructure layer, or a permission problem.
 
At the system level, the four-event observability model fires at every component boundary regardless of transport. Every failed request leaves a trace across the full call chain. The trace shows exactly where the failure occurred, with serialization cost, network cost, and component processing time separately measurable.
 
---
 
### Does this replace DDD, event sourcing, or other architectural patterns?
 
No. Itara does not replace architecture, it concentrates it.
 
You still design your bounded contexts, aggregates, and domain events exactly as before. What changes is where the plumbing lives. Instead of transport configuration, retry policies, and service discovery calls scattered across every service, the communication structure of your system is expressed in one place. The patterns are still yours. The topology that gives them shape is now auditable and changeable without touching code.
 
---
 
### Isn't the wiring config just a configuration file? What makes it architecture?
 
Every system design starts as a diagram: boxes for components, arrows for connections. That diagram is authoritative for exactly as long as it takes someone to start writing code. The moment implementation begins, the diagram becomes a record of an intention, and the code — scattered across HTTP clients, service discovery calls, retry policies, timeout settings — becomes the real, load-bearing truth, whether or not it still agrees with the diagram. Nothing enforces the relationship between the two going forward, which is exactly why architecture diagrams reliably go stale and why understanding a real system means reading all the code, not looking at the drawing that was supposed to describe it.
 
The wiring config is not configuration in the sense of database connection strings or feature flags — values that tune an already-correct system. It's the design, kept alive. The agent doesn't consult it as a reference; it enforces it — an application has no choice but to follow the declared topology, or it doesn't start. Every connection, every deployment boundary, every transport choice is declared there and nowhere else, and the tooling validates it before anything deploys.
 
The difference from a diagram, in one sentence: a diagram can silently stop being true. The wiring config can't — the system has no way to run without agreeing with it.
 
---
 
### Why not just write standard HTTP or gRPC clients?
 
Because the moment you write an HTTP client into a service you make an architectural commitment that is expensive to reverse. Splitting the service later means refactoring the client. Changing which request-reply transport carries the call — HTTP to gRPC, say — means touching both sides.
 
Itara makes that specific decision a configuration entry. Swapping the transport under a request-reply connection — HTTP for gRPC, or for a request-reply mechanism over Kafka — is a wiring config change; the component code never changes because it never knew which transport was carrying the call.
 
This is not a claim that every communication pattern is interchangeable with every other. Moving to genuinely event-driven, publish-subscribe communication is a different exchange pattern, not a transport swap — it changes who knows about whom (a producer doesn't know its consumers), how the APIs need to be written, and how it's expressed in the wiring config as a distinct topology shape (a virtual node), not a transport parameter on an existing connection. Itara makes topology decisions explicit and reversible; it doesn't make every semantic shape equivalent to every other one.
 
---
 
### Does this work with Spring Boot?
 
Yes. Itara and Spring Boot are different layers and coexist naturally — neither knows about the other. A Spring `@Bean` method can call `ComponentLookup` to fetch a component implementation and return it as a bean. Spring manages its context, Itara manages the wiring beneath it. There is no dedicated adapter at this stage — Itara and Spring Boot coexist, but there is no deeper integration.
 
Migrating an existing Spring Boot service is incremental: extract the service interface into a separate API artifact, write an activator (a factory method that returns the implementation), and update the relevant `@Bean` method to pull from `ComponentLookup` instead of constructing directly. The rest of the Spring context is untouched. Deeper integration — for example reusing Spring's servlet infrastructure for HTTP transport — may happen through optional separate libraries in the future.
 
---
 
### Why was Rust chosen as the second implementation language?
 
Two reasons, and they are related.
 
The first is practical: Rust is the planned implementation language for Itara's tooling — the CLI, the deployment tool, and eventually the controller. A working Rust component implementation means the tooling can be tested against real artifacts from day one.
 
The second is philosophical. Rust's central value is that correctness should be a compile-time property — errors caught before the program runs, not at runtime when something breaks in production. Itara's central value for topology is the same thing expressed at the system level. The `.itara` metadata files that every component artifact carries — declared serializer support, contract version information, message format — exist specifically so that tooling can validate the entire system topology before anything deploys. A serializer mismatch, a missing component, an incompatible contract version: these are caught at configuration time, not discovered when the first request fails.
 
The parallel is genuine, not coincidental. A language community that thinks carefully about compile-time correctness is the right audience for a framework that extends that philosophy to distributed system topology.

---
 
### What languages are supported?
 
Java and Rust today, at different levels of maturity. Java has the full feature set — message formats, pluggable authentication and authorization, classloader isolation, everything described elsewhere in this FAQ. Rust is current for what the tooling itself needs — the CLI is a Rust project, and cross-language calls with Java work correctly today, demonstrated int he demo and examples, with a Java component calling a Rust component over HTTP producing a single distributed trace in Kibana. Beyond that, the Rust component/agent implementation is closer to proof-of-concept: it hasn't caught up to Java on more recently shipped capabilities. Go and Python are on the roadmap. The specification is language-neutral; any language with sufficient metaprogramming or build-time automation capability can implement it.
 
---
 
### What happens if two colocated components have conflicting dependency versions?
 
Most of the time, this is solved structurally rather than detected and warned about. Independently developed Java components with conflicting transitive dependencies can be colocated in the same process via a `direct` connection, with no code changes to either. Each component's dependencies are isolated in their own classloader; the shared, common types (contracts, Itara internals) are guaranteed a single class identity across all of them. Two components can depend on different, incompatible versions of most libraries and colocate without conflict, because neither ever sees the other's dependency tree.
 
The real exception is anything that's genuinely a JVM-global singleton rather than an ordinary library — an embedded server (Tomcat and similar), or a logging framework that resolves its own configuration via the thread context classloader (Log4j2, Logback). Two components each trying to initialize their own copy of one of these can conflict regardless of classloader isolation, since the resource they're contending over isn't per-classloader to begin with. These need deliberate placement in the shared classloader rather than being left to isolate automatically — isolation removes the common case of conflict, it doesn't remove the need to think about genuinely global state.
 
Isolated and shared classloader strategies are selectable per deployment — components designed together that already share a dependency strategy can still opt into the simpler shared model.
 
---
 
### How do I handle secrets in the wiring config?
 
The wiring config supports environment variable substitution — any value can be written as `${VAR_NAME}` and the agent resolves it at startup from the environment. This is sufficient for most cases: hostnames, ports, bootstrap server addresses, credentials, and similar connection parameters can all be injected without hardcoding them.
 
More sophisticated secret management — a secret store SPI that can resolve secrets from Vault, AWS Secrets Manager, or similar — is a natural future extension. The substitution pass that already exists provides a clean insertion point for it. For now, env var substitution is the supported and recommended approach.
 
---
 
### Can I use Itara without the CLI?
 
Yes. The CLI is not required to run components — the agent reads the wiring config directly at startup and works without it.
 
What you lose without the CLI is the safety layer: orphaned nodes, version mismatches, connection and node id violations, message-format/serializer compatibility, timeout misconfiguration, and transport capability conflicts are all caught by `itara verify` before deployment. Without it, these surface at startup or at runtime instead. The CLI is not optional polish — it is how Itara keeps its promise that incorrect topologies cannot be deployed silently. The agent alone cannot substitute for it.
 
---
 
### What happens if the wiring config is wrong?
 
Two layers of protection catch configuration errors.
 
The first is the CLI: `itara verify` catches structural errors and compatibility mismatches before anything starts. The set of checks grows with the project and with feedback from real usage.
 
The second is the agent itself: whatever the CLI does not catch, the agent validates at startup time. If a connection cannot be resolved, a referenced artifact is missing, or a configuration is invalid, the agent fails fast and the application does not start. The deployment never succeeds in a broken state. There is no partial startup, no lazy failure on the first call.
 
---
 
### How does Itara interact with service discovery?
 
It doesn't, currently. Itara relies on the infrastructure's ability to resolve the addresses declared in the wiring config — whether that's DNS, a hosts file, or a container orchestrator's internal networking. The address in the config must be resolvable by whatever mechanism the environment provides.
 
A service discovery SPI is on the roadmap, allowing implementations to resolve component addresses dynamically at startup rather than requiring them to be statically declared in the wiring config. It's a smaller priority right now than adoption-focused work — getting-started documentation, authoring tooling, and a versioning story — since those matter more for anyone trying Itara for the first time.
 
---
 
### How do I handle schema evolution in event contracts?
 
Nothing special is implemented at this stage. The usual approaches apply: additive changes are safe, breaking changes require coordinated versioning across producers and consumers. The event contract version declared in the `.itara` metadata file is what the CLI uses to check compatibility — a version bump signals that consumers need to be reviewed.
 
Patterns like consumer-driven contract testing and the expand-contract technique work alongside Itara without any special support. This is an area that will develop further as real-world usage surfaces what tooling is actually needed.
 
---
 
### Should the wiring config be version-controlled?
 
Yes — version controlling the wiring config is the recommended approach. It is the authoritative declaration of the system's topology, and treating it with the same discipline as code is the right instinct.
 
How topology changes are managed across environments, how config versions map to deployments, and how rollback is handled are not yet fully specified. This is active design work, not an afterthought — it's a prerequisite for closed-loop drift detection (comparing what's declared against what's actually running), which is on the roadmap. Feedback on real-world config management needs is welcome.
 
---
 
### What does migration away from Itara look like?
 
Components are plain classes and interfaces. The API artifacts are plain Java interfaces or equivalent. Nothing in the component code requires Itara to compile or run — the only Itara dependency is in the activator, a factory method that wires the component into the topology.
 
Migrating out means replacing the Itara wiring with whatever mechanism you want to use instead, and removing the activator. The business logic, the contracts, and the tests are all untouched.
 
Migration out can also be gradual, the same way migration in can be. Itara coexists with other communication mechanisms without interference — you can move connections out of the wiring config one at a time while the rest of the system continues running through Itara unchanged.
 
---
 
### Where does the name come from?
 
Itara means "of the other" or "belonging to another" in Sanskrit — which fits rather well, since the whole point is that topology belongs to configuration, not to the component. The component does not own its connections.
 
That said, the honest answer is that the core team is not good at naming things. The name came from browsing fantasy, anime, manga, and sci-fi references, merging two names that sounded reasonable together, and checking that the result was not already overused on GitHub. It turned out to also mean something relevant in Sanskrit. That part was a lucky accident.
