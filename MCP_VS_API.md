# MCP and APIs: Decision Framework, Use Cases, and Coexistence Patterns

**A companion analysis to *The Model Context Protocol in 2026* — July 2026**

---

## 0. Executive summary

The question "MCP or API?" is asked more often in 2026 than it is answered well, because the framing is usually wrong. **MCP is not a replacement for APIs; it is a specific consumption pattern layered on top of them.** Nearly every MCP server in production wraps something else — an HTTP API, a database, a filesystem, a subprocess, or all four. The interesting question is not which one to use but *where each is the right shape*, and how they compose in an architecture that has to serve humans, machines, and increasingly capable agents at the same time.

This analysis gives a decision framework, five coexistence patterns, and the concrete criteria that separate "wrap this API as an MCP" from "leave it alone" from "make this MCP-first." It closes with the 2026 twist that is quietly reshaping the tradeoff: sandboxed code execution collapses part of the MCP/API distinction by letting the agent consume MCP-defined contracts *through* direct API calls, which changes what MCP is optimizing for and what it is not.

Bottom-line orientation: **APIs are the substrate; MCP is a lens.** In 2026 the mature architecture uses both — APIs are load-bearing infrastructure that never goes away, MCP is added where it earns its keep (cross-client agent access, discovery, governance chokepoint, curated tool surfaces), and the two are increasingly consumed together via code execution as the token-efficient path.

---

## 1. The framing question is often wrong

The most common way engineers frame this decision — "should we build an API or an MCP?" — treats the two as alternatives. They are not. The confusion is understandable: MCP servers speak HTTP (or stdio), expose invocable operations, and are usually called instead of a REST endpoint from the LLM's point of view, so they *feel* substitutable. But architecturally the substitution is only apparent, not real.

An API defines *what a system can do* and *how any client can invoke it.* It is a general-purpose contract for machine-readable access. Its consumers can be humans through a web app, batch jobs through a script, other services through inter-process calls, and, yes, agents through some integration layer.

MCP defines *how an LLM agent, through a compliant client, can discover and invoke tools, read curated context, and use reusable prompts.* It is a specialized consumption contract designed around the specific ergonomics and constraints of LLM-driven agents: they need tool inventories they can search, descriptions they can read at inference time, consent flows they can surface to users, and results shaped for eventual reasoning over.

Every MCP server has *something* underneath: an API to a SaaS product, a database driver, a filesystem, a shell command. That something is what actually does the work. MCP is one of many possible facades over it — the one specifically designed for agent consumption.

Once the two are seen this way, the question changes from "MCP or API?" to two more useful ones:

1. *Do we need a machine-usable contract at all?* If yes, we need an API (in some form — REST, gRPC, GraphQL, an SDK, a CLI). This is almost always yes.
2. *Do we additionally need an agent-usable facade?* If yes, MCP is the leading candidate for that facade. This is often yes, but not always.

Only if both answers are yes do we build both. Most of this document is about the second question: when the agent facade earns its keep, and what shape it should take.

---

## 2. Anatomy of the two: what each actually is

To decide between them accurately, both artifacts need to be described precisely enough that the comparison makes sense.

### 2.1 API (the general shape)

For this analysis, "API" refers broadly to any machine-callable interface into a system — RESTful HTTP, GraphQL, gRPC, WebSockets, an SDK, a CLI, a database driver. Common properties:

- **Endpoint-level contract.** Each operation is a discrete unit (a REST endpoint, a GraphQL query, a gRPC method, a CLI subcommand).
- **Client-neutral.** Any language, any consumer, any purpose. The API doesn't know or care what will call it.
- **Well-established auth patterns.** OAuth 2.0/2.1, API keys, mTLS, service accounts, JWTs, session cookies. Vast tooling and library support.
- **Precise structured semantics.** Requests and responses are strongly typed; error codes are conventional; retries follow well-known idempotency patterns.
- **Discovery via out-of-band mechanisms.** OpenAPI documents, GraphQL introspection, Protobuf schemas, human-readable docs. Discovery is typically development-time, not runtime.
- **State model varies.** REST tends toward stateless HTTP, gRPC supports streaming, WebSockets are stateful. The API decides what suits its domain.
- **Consumption is programmatic.** The caller must be written to know what endpoints exist and how to compose them. This is a strength (precise control) and a limit (no adaptive discovery at runtime).
- **Ecosystem is decades old, mature, and stable.** REST is standardized; libraries exist for every language; observability and gateway tooling are commodity.

### 2.2 MCP (the specific shape)

MCP is a JSON-RPC 2.0 protocol over stdio or Streamable HTTP with specific primitives (tools, resources, prompts, sampling, roots, elicitation), designed for LLM agent consumption. As of the July 2026 spec:

- **Intent-level contract.** Tools are named actions with descriptions and structured input schemas designed for a model to reason about. Resources are read-only URIs with cache-control metadata. Prompts are macros the client can instantiate.
- **Consumer is an LLM agent through a compliant client.** Every part of the design (tool descriptions in natural language, elicitation flows, consent hooks, resources as context) assumes the caller is a language model with a user in the loop.
- **Auth converging on OAuth 2.1** for HTTP transports as of the 2026-07-28 spec, with stdio still permitted for local subprocess use.
- **Discovery is at runtime and is a first-class feature.** `initialize` handshakes report capabilities; `tools/list`, `resources/list`, `prompts/list` enumerate the surface; Server Cards at `.well-known` endpoints let clients cold-discover before connecting.
- **Result semantics are LLM-friendly.** Tool results are text-first with optional structured content, resource reads carry cache metadata, errors include machine-actionable next steps. The protocol assumes the caller will reason about outputs and choose next actions.
- **Stateless HTTP by default (2026-07-28 spec).** Session state, when needed, is explicit and migration-aware.
- **Composition is server-side.** Prompts as macros let the server define multi-tool workflows the client invokes as one operation.
- **Ecosystem is 18 months old, rapidly maturing, still stabilizing on enterprise features.** SDKs are tiered (SEP-1730). Extensions are landing outside the core spec (MCP Apps, Tasks). Cross-client conformance is high but not universal.

### 2.3 The right mental model

APIs are how systems expose themselves to programs. MCP is how systems expose themselves to *agents*. Programs are precise, opinionated, and single-purpose; agents are adaptive, general, and need help discovering what is available. The two consumer classes have different ergonomic needs, which is why they get different facades.

**Both facades usually cover the same underlying capability.** A Notion API endpoint that creates a page and a Notion MCP tool that creates a page are both facades over the same server-side write operation. What differs is *how* they present themselves, *who* they present themselves to, and *what constraints* the presentation imposes on the caller.

---

## 3. When to use each — a decision framework

Six questions in order. Answer them for a specific capability and the shape of the right facade follows.

### Q1. Who is going to consume this?

- **Machines only** (other services, batch jobs, cron tasks, integrations authored by humans): API is sufficient. MCP adds no value and adds a hop.
- **Humans through a UI**: not the concern of either. A UI calls an API; that is separate from either of these decisions.
- **Human-driven CLI or notebook workflows**: API through an SDK or CLI wrapper. If the workflow is agent-augmented ("Cursor should be able to invoke this from a chat"), MCP becomes relevant.
- **LLM agents through compliant clients** (Claude App, ChatGPT Agents SDK, Cursor, custom LangGraph agent, etc.): MCP is a strong candidate.
- **A mix of all of the above**: build the API layer once; add an MCP facade selectively for the agent-consumed subset.

### Q2. How many distinct clients need to consume it?

- **One client, tightly bound**: API only. If only your in-house Cursor setup needs to invoke it, a plain API called from a code-execution sandbox is the leanest path.
- **Two to three clients you control**: API only if they share a language/stack. MCP facade begins to earn its keep once integration duplication grows.
- **Many clients you don't control** (Anthropic's Claude, OpenAI's ChatGPT, Cursor, Copilot, third-party agents): MCP is nearly always the right answer for the *agent* facade because it inherits the N×M solution the protocol was designed for.

### Q3. Does the caller need runtime discovery?

- **No — caller was written knowing exactly which endpoints exist**: API only.
- **Yes — caller is a general agent that must discover what's available at invocation time, possibly across many servers**: MCP's Server Card, `tools/list`, and `resources/list` are load-bearing for this. Trying to replicate them ad hoc over an API is possible but ends up looking like MCP anyway.

### Q4. Does the caller need per-action consent?

- **No — the caller has been pre-authorized and calls autonomously** (a batch job with a service account): API is sufficient. Add a per-tool authorization matrix if fine-grained control matters, but that's an API-layer concern.
- **Yes — a human is in the loop and should confirm high-impact actions**: MCP's elicitation flow, per-tool permission scoping, and consent primitives are designed for this. Building it over an API means reinventing the same primitives.

### Q5. Do you need a governance chokepoint?

- **No — this system's usage is small or its risk profile is bounded**: API with standard perimeter security is fine.
- **Yes — you want centralized audit, policy enforcement, and rate limiting across all agent-driven usage of many downstream systems**: MCP through a gateway is one of the strongest 2026 patterns for this exact purpose. See the companion case study on the Unified Access MCP.

### Q6. Is latency mission-critical?

- **Yes** (single-digit milliseconds, high-frequency, tight tail latency): API direct. MCP adds at least one hop and, in gateway-fronted architectures, sometimes two. This is fine for the many use cases where 100-500ms is imperceptible, but excludes a real class of low-latency workloads.
- **No**: MCP's latency envelope (typically tens to low hundreds of milliseconds on top of the underlying API call) is invisible to an agent workflow.

### The synthesis

The vast majority of *agent-facing* facades want MCP. The vast majority of *machine-facing* facades want API only. Systems that serve both classes of consumer end up building both, with the MCP layer as a curated projection of the API surface, not a wholesale mirror of it.

---

## 4. Coexistence patterns

Five patterns cover most real-world architectures. Any given system usually falls into one of these; the choice is between them, not between MCP and API.

### 4.1 API-only

**Shape:** the system exposes a REST/gRPC/GraphQL API. Any agents that want to use it consume the API directly through code execution or a bespoke integration.

**When it fits:**
- The system is not agent-facing by design.
- The one or two AI clients that ever call it are internal and can be written to know the API by hand.
- Latency, throughput, or streaming semantics push against MCP's current shape.
- The system is a low-level primitive (a message queue, a cache, a raw data store) that shouldn't be exposed at intent level.

**Trade-off:** cheapest to build and maintain; agent integration requires bespoke code per client; no discovery, no consent flow, no cross-vendor portability.

**Concrete examples:** internal microservices; databases; caches; low-level infrastructure services; ML training pipelines.

### 4.2 API + MCP wrapper

**Shape:** the primary contract is the API. An MCP server sits alongside it, calling the API from its own tool handlers. The MCP surface is a curated subset of the API surface, expressed at intent level rather than endpoint level.

**When it fits:**
- The API already exists and is stable.
- Multiple AI clients want to consume it.
- Agent-friendly tool surfaces (intent-level, small, well-described) differ meaningfully from the raw API (endpoint-level, large, precise).
- You want a governance chokepoint for agent access separate from the machine-to-machine perimeter.

**Trade-off:** two facades to maintain; two auth flows to keep aligned; but the alternative — every AI client writing bespoke integrations — is worse at scale.

**Concrete examples:** SaaS products with mature APIs adding MCP servers: Notion, Linear, Slack, GitHub, Sentry, Stripe, Zapier, HubSpot, Salesforce. In each case the API is the load-bearing contract; MCP is the curated agent facade.

**Key discipline:** the MCP surface should be **outcome-level, not endpoint-level.** A GitHub MCP tool called `create_issue_with_labels(repo, title, body, labels)` is worth more than five tools that separately create the issue, then add each label, then set the milestone. The MCP composes the API calls; the LLM sees one intent, one consent point, one auditable event.

### 4.3 MCP-first, API secondary or absent

**Shape:** the functionality is inherently agent-oriented; there is no meaningful separate API. If a non-agent client ever needs the capability, it uses the same MCP or a thin script that calls it.

**When it fits:**
- The functionality is agent-native — a search index over a corpus that only agents query; a "next daily brief" workflow; a development tool that only exists to be invoked from a coding agent.
- No other machine consumer would ever call the operation, so the API layer would only ever be used by the MCP.
- Building an API separately would introduce duplicate contracts to keep in sync.

**Trade-off:** cross-vendor agent portability is preserved; non-agent usage is inconvenient; some flexibility is lost because you can't call the operation from a non-MCP client easily.

**Concrete examples:** MCP servers for local dev tooling (filesystem, git operations, code search over a repo); domain-specific research agents; the House-style awareness MCPs described in the companion case study.

### 4.4 MCP-as-orchestrator over multiple APIs

**Shape:** an MCP server sits over several APIs (potentially across multiple services or vendors) and exposes composed, workflow-level tools. The composition happens inside the server, not in the LLM.

**When it fits:**
- The valuable agent operation is a composition of several API calls that would be expensive and error-prone to chain through the model's context.
- The composition is stable enough that the server-side workflow definition is worth maintaining.
- Consent, audit, and rate limits should apply at the workflow level, not per API call.

**Trade-off:** server complexity grows; each composed workflow is a piece of business logic that has to be maintained.

**Concrete examples:** "Draft a customer response" that calls a CRM API, a support-ticket API, and a knowledge-base API. "Publish a release" that hits GitHub, a CI pipeline, and a communication channel. Any MCP prompt that runs a multi-step routine as one invocation.

### 4.5 Gateway-fronted composed MCPs (the Unified Access pattern)

**Shape:** many bounded-context MCPs behind one gateway MCP, each of which may itself wrap APIs, databases, or subprocesses. The client sees one MCP; the platform sees several bounded servers, each of which sees the underlying APIs.

**When it fits:**
- Organizations with many backend systems that want unified agent access with centralized governance.
- Enterprises that value auth centralization, audit consolidation, and cross-service trace propagation.

**Trade-off:** most complex to build; strongest governance and consistency; the pattern described in depth in the Unified Access companion.

**Concrete examples:** internal AI platforms fronting product data, HR systems, financial reports, engineering telemetry. Any organization where multiple teams' backends need to be uniformly accessible to agents under one auth and one policy layer.

---

## 5. The 2026 twist: code execution and the compound consumer

The most important recent shift in this comparison is not a change in MCP or in APIs; it is a change in *how agents consume both.*

Through 2025 the assumed consumption model was: the agent maintains all tools in its context, decides which to call, receives the result in its context, and reasons over it. This model is what MCP was designed against. It is also what made MCP expensive in tokens.

The 2026 code-execution pattern (Anthropic's engineering post; Cloudflare's "Code Mode"; wide replication since) changes the consumption model. The agent has a sandbox with a filesystem and a scripting environment. MCP tools appear as files or modules; the agent writes code that calls them; only the final result returns to the model's context.

This shift has three consequences for the MCP-vs-API decision:

**1. The tool schema becomes documentation rather than an active context load.** In the classic model, every tool definition sat in the LLM's window for every query. In the code-execution model, tool definitions live in the sandbox; the agent reads them on demand. This slashes the token cost of a large MCP surface — the same expense that made "just build a big MCP" prohibitive in 2025.

**2. Direct API consumption becomes practical for sandboxed agents.** If the agent can write code, it can call an HTTP API directly with the same effort as calling an MCP tool. For many workloads the direct call is cheaper: fewer hops, richer result handling, and no MCP protocol overhead. The paper's Section 5.4 debate about "skills vs MCP" is exactly this. For local dev tooling where the sandbox and the tool live on the same machine, direct API/CLI often wins.

**3. MCP shifts from "protocol the model calls at inference time" toward "protocol the model programs against."** The Server Card becomes the interface contract the agent's code is written to. The tool definitions become the API surface. The consumption is through code that the agent writes on the fly. MCP's role becomes closer to a standardized documentation and discovery layer than to a live invocation channel.

The mature synthesis: MCP wins where cross-client, cross-vendor, governed access matters (the protocol becomes the shared contract regardless of who's consuming it). Direct API consumption wins where a single sandboxed agent needs the shortest path to the underlying system (efficient, deterministic, no protocol overhead). Increasingly, both live in the same architecture, with MCP providing the *discovery and governance* layer over APIs that the sandboxed agent may eventually call directly through its code environment.

This is why the framing "MCP or API?" is losing coherence in 2026. In many mature architectures the answer is both, consumed together: **the MCP defines the contract; the API is the runtime; the sandbox is where composition happens.**

---

## 6. Best practices

### 6.1 For APIs that will be agent-consumed (whether via MCP or directly)

1. **Version explicitly.** Agent workflows break silently on API changes; explicit versioning makes it recoverable. `/v1`, `/v2`, header-based versioning — pick one and hold the line.
2. **Return structured errors with next-step guidance.** Errors that only tell the caller what broke are hostile to LLM consumption; errors that suggest retries, alternative tools, or user prompts are usable directly by agents. This is a good practice for any API, but especially important when an agent is on the other end.
3. **Support idempotency keys on mutations.** Agents retry aggressively and parallelize. An API that produces double-effects on retries will produce them constantly.
4. **Pagination via opaque cursors, not offset+limit.** Cursors are stable across concurrent writes; offsets are not, and agents will hit the race.
5. **Emit standard trace headers.** W3C Trace Context (`traceparent`, `tracestate`, `baggage`) is now the interchange standard. Any API that will be composed with MCP through a gateway needs to propagate these or the whole distributed-trace story falls over at that hop.
6. **Design rate limits with agent behavior in mind.** Agents burst. A rate limit that assumes steady human usage will misfire; account for parallel retries and provide clear `Retry-After` headers.
7. **Keep authentication tokens narrowly scoped.** If an MCP gateway will be calling this API on behalf of many agents, the tokens issued for that path should be scoped as tightly as possible — audience-bound, per-tool, short-lived.

### 6.2 For MCP servers that wrap APIs

8. **Design outcome-level tools, not endpoint-per-tool wrappers.** The single most common failure mode of an "API + MCP wrapper" is mapping every endpoint one-to-one. The MCP surface should be a *curated projection* of the API — small, intent-level, LLM-friendly.
9. **Keep the tool count small.** 5-8 tools per server is the empirical sweet spot; 15+ degrades model choice-making measurably. If your API has 40 endpoints, do not build 40 tools.
10. **Do not pass caller tokens through to the API.** Terminate at the MCP; call the API with server-side credentials scoped and audit-tagged with the caller identity. This closes the confused-deputy vector.
11. **Cache thoughtfully.** Use the 2026 `ttlMs`/`cacheScope` on resource reads; cache the API's own responses server-side where semantics permit; do not cache authenticated responses across users.
12. **Idempotency at both layers.** The MCP tool and the underlying API call should both support idempotency; the MCP tool should propagate a client-generated ID all the way through.
13. **Aggressively map API error semantics into MCP-friendly errors.** A raw `429` or `503` from the API is usable, but an MCP error with `retry_after`, an explanation, and a suggested next action is dramatically better for agent behavior.
14. **Document the underlying API in the server-side prompt.** If the LLM needs to reason about *why* a tool exists, the prompt or a resource can hold the API rationale, contract stability guarantees, and known quirks. This is under-invested in most MCP servers and pays back immediately.
15. **Publish the Server Card.** Discoverability is a first-class feature of MCP; skipping the `.well-known` endpoint wastes half of what the protocol offers.

### 6.3 For architectures that use both

16. **Choose one as the load-bearing contract.** Usually the API. The MCP is a facade; it should never define semantics the API cannot support.
17. **Don't drift the two surfaces.** Every capability exposed on the MCP should trace back to a defined API operation. Do not add hidden business logic that lives only in the MCP tool handler; that logic belongs in a service the API can call too.
18. **Unify observability.** Both facades should feed the same trace, log, and audit pipelines. Two parallel observability trees is one too many.
19. **Decide on the auth model early.** Is the MCP a public front-door (OAuth 2.1 against the org's identity provider), or is it an internal facade over an internal API (a service-to-service auth pattern)? Either is legitimate; mixing them incoherently is not.
20. **Version the MCP tool surface along with the API surface.** A breaking API change usually implies a breaking MCP change. Coordinate.

### 6.4 For code-execution-enabled agents

21. **Publish the Server Card in a form the sandbox can consume.** When agents write code against your MCP, they need the schema in a machine-readable form — OpenAPI-adjacent JSON or SDK-like Python/TypeScript is far more useful than tool descriptions alone.
22. **Provide typed SDKs for common languages.** The 2026 pattern of "agent writes code against your MCP" is easier when the MCP's contract is expressible as an SDK type. Do not make sandboxed agents parse Server Cards by hand.
23. **Assume both invocation modes.** Some clients call your MCP tools directly; some call the underlying API directly from a sandbox; some do both. Design your rate limits, audit trails, and identity flows to handle both cleanly.

---

## 7. What to avoid

### 7.1 Anti-patterns for API-only teams facing agent adoption

**Ignoring the agent consumer.** Assuming your API is "just" for machines and refusing to design for agents produces two symptoms: agents call your API awkwardly through bespoke integrations, and the eventual MCP wrapper (built by someone else, badly) becomes a permanent liability. Even without MCP, agent-friendly API design (structured errors, idempotency, discoverable schemas) is worth doing.

**Trying to make the API "smarter" to compensate for the lack of an agent facade.** Adding a natural-language endpoint, a plain-English wrapper, or a chat-style query interface to your existing REST API in an attempt to serve agents. This produces a poorly-shaped facade in the wrong place. If you want to serve agents, build an MCP; do not distort your API.

**Auth divergence.** Different auth schemes for different consumer classes (API key for humans, OAuth for MCP) that share the same underlying capability. This produces double auditing surfaces and inevitable drift. Consolidate on OAuth 2.1 and scope tokens appropriately.

### 7.2 Anti-patterns for MCP-only teams

**Assuming MCP replaces the API layer.** Building an MCP without a coherent underlying API means you have committed to MCP being the only way to invoke your capability. This is fine for genuinely agent-native functionality; it is a costly mistake for anything else. Non-MCP consumers appear eventually.

**God-server.** 30+ tools in a single namespace. Covered in the paper and the case study; called out here because it is the most common way MCP-first teams overreach.

**Endpoint-per-tool wrapping over an unstated API.** If you find yourself writing thin tools that each hit one internal function directly, you have an implicit API. Extract it, define it, and put the MCP as a facade over it. Otherwise you can never expose the same capability to a non-agent consumer without duplicating the work.

**Building for the wrong client population.** Building an MCP for a capability that will only ever be consumed by one client is wasted effort. Use the API; call it from the sandbox; skip the protocol overhead.

### 7.3 Anti-patterns for teams running both

**Two facades that drift.** The API gets a new capability; the MCP doesn't. Or the MCP tool has business logic the API doesn't. Or the error semantics diverge. This is the single most common failure of hybrid architectures. Automate the discipline: every API change requires an MCP audit; every MCP tool traces back to an API operation.

**Two auth flows without a shared identity source.** Users authenticate against one identity provider for API access and a different one for MCP access. Confusion follows. Consolidate on one identity source; issue different token scopes for different consumers.

**Two observability trees.** Traces don't connect across the MCP-to-API boundary because the two surfaces log independently. Fix at the trace-context level (SEP-414 propagation) and at the audit-log level (one pipeline, both feeds).

**MCP as a way to bypass API governance.** The API has strict rate limits and requires a compliance review to add capabilities; the MCP is treated as a fast-lane for the same capabilities without those checks. This is a governance failure; the fast-lane will be the source of the next incident.

**Chaining an MCP through another MCP through an API "because MCP is the standard."** Every layer adds latency and a failure mode. If the API can be called directly for a use case, calling it directly is often the right answer.

### 7.4 Anti-patterns specific to sandboxed / code-execution consumption

**Making the MCP the *only* consumption path.** A sandbox with code execution can often call the API directly. Forcing every call through MCP wastes tokens and adds latency. Let the agent decide; provide both.

**Not publishing typed SDKs.** Expecting sandboxed agents to parse Server Cards and construct raw JSON-RPC calls when a Python or TypeScript SDK would suffice. This is a solved problem; solve it.

**Ignoring the fact that direct API calls bypass MCP governance.** If your governance model requires every agent call to go through the MCP, and your sandboxed agent can bypass the MCP by calling the API directly, your governance model is a wish. Either enforce it at the API layer (network policy, service-to-service auth) or accept that MCP is a soft governance layer.

---

## 8. Reference decisions — concrete organization scenarios

Working through the framework on real shapes.

### Scenario A: A SaaS product with a mature REST API adds agent support

Existing state: REST API used by web app, mobile clients, integrations. Now customers want to invoke it from ChatGPT, Claude, Cursor, and internal agents.

**Decision:** API + MCP wrapper (pattern 4.2).

**Rationale:** the API is load-bearing; every non-agent consumer already depends on it. The agent consumer class is real and multiplying. The MCP wrapper curates a small outcome-level tool surface, terminates OAuth at its own layer, and delegates to the API with server-side credentials. The wrapper is a small ongoing maintenance cost that pays off once more than two AI clients are in the wild.

**Not:** MCP-first (would abandon existing consumers); API-only (leaves cross-client agent adoption as everyone's problem).

### Scenario B: An internal data platform team building AI-native tools for their organization

Existing state: internal analytics stack, vector databases, log stores, entity registries — all consumed today by data scientists via SDKs and notebooks. No external consumers. Team wants unified agent access with strong governance.

**Decision:** Gateway-fronted composed MCPs (pattern 4.5), i.e., the Unified Access MCP.

**Rationale:** the audience is agents through many clients; the systems behind are numerous; the governance requirement is real. This is exactly the pattern the companion case study covers.

**Not:** API-only (leaves every agent to build bespoke integrations); one big MCP (god-server anti-pattern).

### Scenario C: A trading firm evaluating MCP for market-data access

Existing state: proprietary market-data pipelines, low-latency feeds, tick-level data consumed by trading algorithms.

**Decision:** API-only (pattern 4.1). Selectively an MCP wrapper for analyst-facing tooling (research, backtests) that agents may drive, kept strictly out of the low-latency path.

**Rationale:** the mission-critical path cannot afford the MCP hop. But research and analyst workflows can benefit from an MCP facade. Split the surface: two facades for two audiences.

**Not:** MCP wrapping the tick feed (adds latency for zero benefit); API-only for everything (misses the analyst-agent value).

### Scenario D: A dev tool startup building an agent-native code review assistant

Existing state: greenfield. The tool is agent-oriented from day one.

**Decision:** MCP-first (pattern 4.3). If a non-agent consumer eventually appears, expose the same capability through a thin API. Do not build both facades until forced to.

**Rationale:** the audience is agents; there is no existing API to preserve; building both facades pre-emptively is speculative work. Ship the MCP; add the API on demand.

**Not:** API + MCP wrapper (over-engineered for greenfield); ignoring MCP entirely (misses the leading distribution channel for agent-facing tools in 2026).

### Scenario E: An enterprise integrating MCP with an existing API gateway

Existing state: mature API gateway (Kong, Apigee, AWS API Gateway) fronting dozens of internal services. Now they want agent access with governance.

**Decision:** MCP as a governed facade behind (or beside) the API gateway. Route MCP through the same identity provider, the same audit pipeline, the same rate-limit fabric. Consider MCP-Gateway products (Docker MCP Gateway, Cloudflare's gateway) as the MCP-specific layer, backed by the API gateway for downstream service calls.

**Rationale:** enterprise governance is already solved at the API gateway; the MCP layer inherits it. Reinventing at the MCP layer is wasted effort and, worse, produces two governance surfaces that will drift.

**Not:** MCP as a governance island; direct API access bypassing the API gateway (would fragment audit and identity).

### Scenario F: A team with heavy code-execution-based agents (Claude Code, Cursor, custom agent with sandbox)

Existing state: agents run with sandbox access; they call MCP servers for some things, CLI/API for others.

**Decision:** Publish the tool surface via MCP for discovery and governance; publish an SDK / OpenAPI for direct invocation from the sandbox. Let the agent choose.

**Rationale:** the sandboxed agent will sometimes prefer direct API calls (efficiency) and sometimes prefer MCP (governance, cross-client consistency). Serving both consumption modes is cheaper than forcing one, and the 2026 landscape strongly rewards this compound consumer.

**Not:** MCP-only (blocks efficient sandbox consumption); API-only (loses cross-client agent portability).

---

## 9. The economic layer

A dimension often left implicit in these comparisons: money.

**APIs price at the per-call layer.** Metering is well-understood; caching helps; rate limiting is the discipline. Costs scale with volume and are predictable.

**MCP adds two economic effects:**
1. *Model tokens consumed by tool definitions and results.* The classic model loads all tool defs upfront; a 40-tool MCP surface can cost tens of thousands of tokens per query. Lazy loading and tool search mitigate; code execution largely eliminates.
2. *Underlying API calls remain metered at the API layer.* MCP doesn't reduce API cost; it may increase it slightly through overhead and retries.

**In hybrid architectures:**
- Serving both an API and an MCP means both cost structures apply.
- Rate limits should be aligned across facades so agents can't accidentally break themselves by bursting through one path.
- Cost attribution needs a common identity model; otherwise, "which agent cost us that much?" becomes unanswerable.

**In code-execution architectures:**
- The classic MCP token cost mostly disappears (defs read on demand, results filtered before returning to context).
- The API cost dominates (each direct call from the sandbox is a real API call).
- Total spend often *decreases* compared to classic MCP consumption at scale — a factor of 10× has been reported by well-instrumented teams.

The economic verdict: pure MCP consumption is the most expensive path per query. Pure API consumption is the cheapest. MCP + code execution is close to API-only cost with the discovery and governance benefits of MCP added. The mature 2026 architecture optimizes for the third.

---

## 10. Verdict

Three points to remember when the "MCP vs API" question comes up next quarter.

**1. It is rarely one or the other.** APIs are the substrate; MCP is the lens for agent consumption. Every serious MCP has an API-shaped something underneath. Every serious API deserves an MCP facade *if* it will be agent-consumed by more than one client. The interesting question is where each layer earns its keep.

**2. The consumer decides the shape.** Machine-to-machine and human-driven flows want APIs. LLM-driven agent flows through many clients want MCP. Systems that serve both build both, disciplined.

**3. Code execution collapses part of the distinction.** In 2026 sandboxed agents can consume MCP-defined contracts through direct API calls, keeping the discovery and governance value of MCP while paying near-API-only tokens. This is the strongest cost model available. Build for it: publish Server Cards, publish typed SDKs, let the agent choose the path.

The organizations that get this right treat MCP and API as complementary facades over the same underlying capabilities, discipline the two to stay in sync, and consume them through whichever path each specific agent workflow needs. The organizations that get it wrong treat them as competitors, pick one prematurely, and end up rebuilding the other in a distorted shape three quarters later.

The 2026 answer is not either/or. It is: **API always; MCP where it earns its keep; code execution where efficiency matters; all three governed as one system.**
