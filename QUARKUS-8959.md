# QUARKUS-8959 - MCP client according to 2026-07-28

JIRA: https://redhat.atlassian.net/browse/QUARKUS-8959

Upstream PRs: 
- https://github.com/langchain4j/langchain4j/pull/5881
- https://github.com/quarkiverse/quarkus-langchain4j/pull/2723

Follow-ups: 
- https://github.com/quarkiverse/quarkus-langchain4j/pull/2922
- https://github.com/quarkiverse/quarkus-langchain4j/pull/2968

Documentation: https://docs.quarkiverse.io/quarkus-langchain4j/dev/mcp.html

Related plans: 
- [QUARKUS-8895](QUARKUS-8895.md) Include Quarkus MCP Server 2.x in IBQ 3.40
- [QUARKUS-7049](QUARKUS-7049.md) Full support for MCP Client

LangChain4j's MCP client now implements MCP revision `2026-07-28` next to `2025-11-25`. The new revision is stateless:
no `initialize` handshake and no `Mcp-Session-Id`. The client probes the server with `server/discover`, and every
request carries its own protocol version. Quarkus consumes this through the LangChain4j client and its stdio and
WebSocket transports, plus its own HTTP transport.

## Scope of the testing

In scope:

* Default auto-detection on stdio, HTTP and WebSocket against modern and legacy-only servers, plus silent servers
  (HTTP) and unknown-method-intolerant servers (stdio and HTTP).
* Detection failure on authentication.
* Request headers sent by the Quarkus HTTP transport: protocol headers, `Mcp-Param-*`, custom headers, TLS registry.
* Subscriptions on stdio and WebSocket (HTTP is already covered upstream), including unsubscribe, list-change flags and
  the removed methods.
* Tool timeout and cancellation on stdio and HTTP. WebSocket is excluded because it cancels the same way as
  stdio.
* Mixed-version clients in one application.
* Retry of multi-round-trip (`input_required`) responses.

Out of scope:

* Server behavior and the general `2026-07-28` functional matrix, covered by [QUARKUS-8895](QUARKUS-8895.md).
* `DockerMcpTransport` (no Quarkus `docker` transport), `langchain4j-agentic-mcp` (not exposed), upstream DTO,
  header-encoding and schema unit tests.
* Tracing of the new operations, health check and metrics.
* Servers outside the Quarkus MCP Server family (e.g. `npx` reference servers), beyond the unknown-method case in
  scenario 1.

## Extension interactions and integration points

* **Quarkus MCP Server 2.x**, as included in the platform (2.0.4): it serves both revisions, so a client
  pinned to `2025-11-25` is the legacy case.
* **OIDC / client authentication provider** — with no session, the provider is consulted on every POST, including the 
  probe, so auth failures surface during detection, not at the first tool call.
* **TLS registry** — applies to HTTP and WebSocket. Upstream TLS tests are legacy only.
* **Custom headers** (`header.*` and the headers supplier) — merged with the new protocol headers in one place.
* **AI services / `McpToolProvider`** — `Mcp-Param` headers need `listTools()` before `executeTool()`, which the tool
  provider does.

## Getting familiar with the feature

* Reviewed the upstream PR and tutorial, `DefaultMcpClient` (detection, `autoDetect`, multi-round-trip loop,
  subscriptions, cancellation), `ToolSpecificationHelper`, `McpHeaderEncoding` and the transport additions.
* Reviewed the Quarkus integration (`McpRecorder`, `QuarkusStreamableHttpMcpTransport`, runtime config,
  `integration-tests/mcp`, layout of `mcp-conformance`), the QE `ai/mcp` module and the related plans.
* Quarkus maps `protocol-version` and the three `subscribe-to-*-list-changes` flags; `protocolDetectionTimeout` and
  `multiRoundTripMaxRetries` are not exposed.
* Version under test: the Quarkus platform `3.40` branch ships Quarkus LangChain4j 1.13.4 (LangChain4j 1.19.3, with the
  modern HTTP transport) and Quarkus MCP Server 2.0.4. Upstream `main` is on LangChain4j 1.22.0.

## Existing test coverage

Upstream LangChain4j has extensive unit tests with mocked transports and `*IT` classes in modern and `Legacy`
variants over all three transports. Transport behavior is asserted against LangChain4j's own HTTP transport, not the
Quarkus one.

Upstream Quarkus LangChain4j (`integration-tests/mcp`, `mcp/deployment`) was split into modern and `Legacy` classes, 
with the following gaps:

* Auto-detection is tested only over HTTP; no silent-server test.
* TLS, static and dynamic header tests exist only as `Legacy` classes.
* Modern subscriptions are tested only over HTTP; none over stdio or WebSocket.
* No tests for the deprecated methods throwing, `subscribe-to-*-list-changes=false`, or a tool timeout on the modern
  protocol.
* `Mcp-Param` tests miss non-ASCII values, a missing prior `listTools()` and invalid annotations.

QE `ai/mcp` builds a real server app and a real client app per test with the QE framework against the platform BOMs, so 
it uses Quarkus MCP Server 2.0.4 and the platform's LangChain4j. It exercises tools, resources and prompts over the
three transports, with the following gaps:

* No `protocol-version` is set, so the client already auto-detects against a modern server; the modern case is covered
  implicitly. There is no legacy-only, silent or intolerant server, and no legacy fixture or `Legacy` class pairs.
* No tests for subscriptions, list-change flags, request headers, TLS, timeouts or multi-round-trip.
* Stdio runs the server jar with `java -jar`; the framework cannot stop it cleanly and kills it on timeout.

This plan adds to that module and to the [QUARKUS-8895](QUARKUS-8895.md) work (every client path pinned to a revision, 
the `2026-07-28` functional matrix and server-side checks). It does not repeat them; it covers only what depends on the
client's own behavior: unpinned detection, what the client sends, timeouts and cancellation, and retries.

## Planned coverage

All scenarios go into the existing `ai/mcp` module.

1. **Default protocol detection** (no `protocol-version`). *Why:* the only change that affects applications that did
   nothing, and a silent legacy server costs the full detection timeout on every startup.
   - Real server and legacy-only stub (all three transports): the 2.x server ends up modern; the stub, which
     speaks only `initialize`, `tools/list` and `tools/call`, makes the client fall back and still list and call tools.
   - Silent server (stub, HTTP only): `server/discover` is ignored, `initialize` and `tools/list` answered.
     With `initialization-timeout` at a few seconds, assert the fallback warning and a startup delay of about one
     timeout. Variant where `initialize` is also ignored: assert the failure message and a delay of about 2×.
   - Intolerant servers (stub, stdio and HTTP): a stdio process that exits on `server/discover`, and an HTTP
     stub that closes the connection or answers 5xx. Assert how the failure reads, and that
     `protocol-version=2025-11-25` restores the connection. A third-party HTTP server is an optional extra.
   - Mixed versions: one client pinned to `2025-11-25` and one auto-detected, both against the 2.x server. Both tool 
     sets are listed and callable; client keys stay distinct in logs.

2. **Detection failure on authentication** — modern server requiring a bearer token; client with none or an invalid one.
   A failed probe that is not a protocol-level rejection falls through to the legacy `initialize`, so the user may see
   a generic failure instead of the authentication problem.
   - No token or an invalid token: the startup failure and the logs show the authentication cause (unauthorized), not
     only a generic "answered neither the detection request nor the initialization".
   - A valid token from the named OIDC client: detection succeeds and the client
     ends up modern, which shows the probe is authorized too.
   - A token accepted for `initialize` but rejected on `server/discover` (for example a scope mismatch): the fallback
     does not hide the authorization problem.
   If the cause is not visible, record a finding for upstream.

3. **Quarkus HTTP transport headers** (`2026-07-28`, a server tool that echoes the request headers):
   - Protocol headers: `MCP-Protocol-Version`, `Mcp-Method`, `Mcp-Name` for `tools/call`, `resources/read` and
     `prompts/get`.
   - `Mcp-Param-*`: non-ASCII (`=?base64?...?=`), leading whitespace, nested parameter; `executeTool` without a
     prior `listTools()` omits headers with a warning and still succeeds; a tool with an invalid `x-mcp-header`
     (non-token name, duplicate, forbidden type) is absent from `listTools()` without hiding valid ones.
   - Custom headers and TLS: `header.*` and the headers supplier arrive alongside, not instead of, the protocol
     headers. TLS registry client against a TLS server over HTTP, and over `wss://`.

4. **Subscriptions** on stdio and WebSocket (HTTP exists): `subscribeToResources` with several URIs delivers
   `McpResourceUpdatedEvent` and stops after `unsubscribeFromResources` (stdio uses `notifications/cancelled`, HTTP
   closes the stream). Deprecated methods and `setRoots` throw on a modern server; `subscribeToResources` throws on a
   legacy one. With `subscribe-to-tool-list-changes=false` (and prompt/resource) `listTools()` returns the cached
   list after a server change; with `true` it refreshes.

5. **Timeout and cancellation** — a tool sleeping longer than `tool-execution-timeout`, on stdio and HTTP with
   `2026-07-28` and HTTP with `2025-11-25`. The client returns the timeout result and a follow-up call succeeds. The
   tool records whether its sleep was interrupted (flag read through a second tool), asserted within a bounded time.
   If 2.x does not interrupt on cancellation, record a finding rather than weakening the assertion.

6. **Multi-round-trip retry** — if 2.x can emit `input_required` with only `requestState`, assert the retry completes
   the call; otherwise use the stub: `requestState` only (succeeds), `inputRequests` present (fails with the clear
   "cannot handle" message, not a hang), never-complete (fails after 3 retries). The `inputRequests` limit is not in the
   upstream tutorial or the Quarkus guide: record it as a finding.

## Impact on test suites and testing automation

* No new module or CI job. New `*IT` classes go into `ai/mcp` next to the existing ones, reusing `BasicMCPIT` and the
  REST client app, extended with endpoints for the new operations (subscribe, headers echo, timeout tool).
* One stub, in the test sources and used by scenarios 1, 3 and 6, that plays a legacy-only server (answers `initialize`,
  `tools/list`, `tools/call` and returns method-not-found for the rest) and a misbehaving one (ignores `server/discover`
  or `initialize`, closes the connection, emits invalid `x-mcp-header` schemas or `input_required`).

## Impact on resources

The added tests should have **low to moderate impact** on resources:

* An estimated execution time increase of 5 to 6 minutes.
* Tests will be executed on baremetal and OpenShift (only for classes without the stub), in JVM and native mode.

## Contacts

* Tester: Roberto Cortez <radcortez@ibm.com>

## References

* https://docs.quarkiverse.io/quarkus-langchain4j/dev/mcp.html
* https://github.com/langchain4j/langchain4j/pull/5881
* https://github.com/quarkiverse/quarkus-langchain4j/pull/2723
* https://github.com/quarkiverse/quarkus-langchain4j/pull/2922
* https://modelcontextprotocol.io/specification/2026-07-28
* https://modelcontextprotocol.io/specification/2025-11-25
* [QUARKUS-8895](QUARKUS-8895.md)
* [QUARKUS-7049](QUARKUS-7049.md)
