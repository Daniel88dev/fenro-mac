# Running an MCP server inside a sandboxed macOS app

Research for the wayfinder ticket "Research running an MCP server inside a sandboxed macOS app". Facts were gathered on 2026-09-12 from the MCP specification, the Swift MCP SDK repository, Claude Code documentation, and Apple documentation and Developer Forums posts by Apple staff. Items not confirmed by a primary source are marked UNVERIFIED. This document surfaces facts and a recommendation; the decision is made in the ticket "MCP transport and lifecycle decision". Assumes App Sandbox on and direct notarized distribution.

## Summary

- **Claude Code** connects to local servers over **stdio** (it launches a process) or **Streamable HTTP** (it POSTs to a URL). It can exec any binary, including one inside Fenro's app bundle. Claude Desktop, by contrast, only works with stdio servers in practice.
- A sandboxed app **can** listen on 127.0.0.1 with the `com.apple.security.network.server` entitlement. No privacy prompt is involved; the macOS firewall may prompt once if the user has it on and has disabled auto-allow for signed software.
- The **official Swift MCP SDK** (0.12.1, May 2026) gives a Swift 6 `actor Server` with tools, resources and prompts and a spec-compliant Streamable HTTP *adapter*, but **no HTTP listener**: you bring your own tiny HTTP server. It is Tier 3 (experimental), implements spec 2025-11-25, and has had no commits since April 2026. **Cocoanetics/SwiftMCP** is an active alternative with a built-in HTTP server and macros.
- The MCP spec had a **breaking revision on 2026-07-28** (stateless, no `initialize`). Claude Code probes HTTP servers for it and falls back to the legacy handshake, so a legacy-only server still works today, but must reject the probe cleanly.
- The **stdio shim** pattern (a small bundled binary Claude Code launches, forwarding to the running app) is exactly what Xcode 26 does with its `mcpbridge`, though Apple uses private APIs for the hop. A third-party shim forwards over loopback HTTP or a Unix socket in an App Group container. The shim can also launch the app if it is not running.
- **Recommendation**: host Streamable HTTP on loopback inside the app, and ship a thin stdio shim that proxies to it. HTTP clients connect directly; stdio-only clients and auto-launch go through the shim. Details and flip conditions at the end.

## 1. Swift MCP SDK state

Source repository: https://github.com/modelcontextprotocol/swift-sdk

- **Releases**: 0.12.1 (2026-05-07), 0.12.0 (2026-03-24, auth per SEP-990), 0.11.0 (2026-02-19, spec 2025-11-25 coverage, server HTTP transport added), 0.10.2 (2025-09-23, strict concurrency). Last commit on `main` 2026-04-29. Pre-1.0: minor versions may break.
- **Protocol versions** supported: 2025-11-25, 2025-06-18, 2025-03-26, 2024-11-05. The 2026-07-28 revision is not implemented; tracking issues are open. Source: `Sources/MCP/Base/Versioning.swift`.
- **Server API**: `public actor Server` with `withMethodHandler`, capabilities for tools, resources (subscribe, listChanged), prompts, completions, logging; server-to-client sampling and elicitation; `Server.currentHandlerContext?.httpContext` exposes request headers inside a handler (useful for a token check). Everything is `Sendable` and strict-concurrency clean; Swift tools 6.1, macOS 13+.
- **Transports**: `StdioTransport` (client or server), `HTTPClientTransport` (client only), `InMemoryTransport`, `NetworkTransport` (wraps one existing `NWConnection`, no listener support), and `StatefulHTTPServerTransport` / `StatelessHTTPServerTransport`. The two server transports are **framework-agnostic adapters**: you call `handleRequest(_ request: HTTPRequest) async -> HTTPResponse` from your own HTTP server and translate the result, including streaming SSE bodies. The in-repo example of this wiring is the conformance server built on swift-nio (`Sources/MCPConformance/Server/HTTPApp.swift`), which is not published as reusable API. swift-nio is a dependency only of that executable, not of the `MCP` library.
- Included validators: `OriginValidator.localhost(port:)`, `AcceptHeaderValidator`, `ContentTypeValidator`, `ProtocolVersionValidator`, `SessionValidator`. There is **no static bearer-token validator**; issue #284 requests one. A token check is a few lines in a handler using `httpContext`.
- Caution: context7's generated docs for this SDK show a `StatefulHTTPServerTransport(port:host:)` initializer that does not exist in the source.
- **Status**: listed as Tier 3 ("experimental, partially implemented") on https://modelcontextprotocol.io/docs/sdk. 107 open issues, several unreviewed correctness PRs (stateless transport context leakage #265, stdio frame interleaving #263, NetworkTransport socket leak #282). New maintainers from MacPaw took over in January 2026 after a gap.
- **Sandbox**: issue #150 confirms a sandboxed app worked after enabling both incoming and outgoing network entitlements.

### Alternatives

- **Cocoanetics/SwiftMCP** (https://github.com/Cocoanetics/SwiftMCP): macro-based (`@MCPServer`, `@MCPTool`), built-in NIO HTTP server with Streamable HTTP at `/mcp`, stdio and TCP transports, OAuth/JWT support. v1.10.4 on 2026-08-14, very active. Heavier dependency tree (swift-nio, swift-syntax, swift-crypto). Spec version not stated in its README, UNVERIFIED.
- **stallent/hummingbird-mcp**: Hummingbird 2 adapter, but depends on a fork and predates the official server transport; likely stale.
- Writing a minimal loopback HTTP/1.1 server on `NWListener` that feeds the official SDK's `handleRequest` is a few hundred lines; the conformance server shows the shape.

## 2. What Claude Code supports and how it is configured

Source: https://code.claude.com/docs/en/mcp.md and https://code.claude.com/docs/en/cli-reference.md

- **Transports**: `http` (Streamable HTTP), `stdio`, `sse` (deprecated), `ws` (via JSON only).
- **Adding a server**:
  ```bash
  claude mcp add --transport http fenro http://127.0.0.1:47777/mcp --header "Authorization: Bearer <token>"
  ```
  ```bash
  claude mcp add --transport stdio fenro -- /Applications/Fenro.app/Contents/MacOS/fenro-mcp
  ```
  or `claude mcp add-json fenro '{"type":"http","url":"...","headers":{...}}'`. Scopes: `local` (default, stored in `~/.claude.json` per project path), `user` (`~/.claude.json`), `project` (`.mcp.json` in the repo, prompts for approval in interactive sessions).
- `.mcp.json` supports `${VAR}` and `${VAR:-default}` expansion in `url`, `headers`, `command`, `args`, `env`. Claude Code sets `CLAUDE_PROJECT_DIR` in a stdio server's environment, which is how a stdio shim can learn the caller's working directory for Project resolution. Over HTTP there is no such signal; the client would have to pass a project explicitly or the tool would need another cue.
- **Static bearer header** works; if the server rejects it Claude Code reports failure rather than falling back to OAuth. OAuth is also supported (`/mcp`, `claude mcp login`).
- **Protocol negotiation**: Claude Code 2.1.232+ runs the MCP TypeScript SDK 2.0 runtime and "asks HTTP servers whether they support the newer revision" (2026-07-28), falling back to the legacy `initialize` handshake otherwise. For stdio servers it only probes if `MCP_PROTOCOL_NEGOTIATION=auto`. A legacy-only HTTP server must answer the modern probe with a clean 4xx or a recognised error so the fallback triggers. UNVERIFIED: the exact request the probe sends.
- **Connection behaviour**: first connection retries 3 times on connection refused or 5xx; mid-session drops use exponential backoff up to 5 attempts; startup timeout `MCP_TIMEOUT` default 30 s; tool-idle timeout 5 min for HTTP, 30 min for stdio.
- **Permissions**: rules use `mcp__fenro` or `mcp__fenro__tool_name`. A server can force a confirmation prompt per tool with `_meta["anthropic/requiresUserInteraction"]`.
- **No MCP install deep link exists.** Claude Code's `claude-cli://open` scheme only takes `q`, `cwd`, `repo`. The app's options are: show the one-line command, shell out to `claude mcp add-json`, or write `.mcp.json` / `~/.claude.json` itself.
- **Claude Desktop**: official docs describe only stdio (`claude_desktop_config.json` with `command`/`args`) and MCPB bundles (stdio only, no HTTP server type). A GitHub issue reporting Desktop stripping a localhost `url` entry was closed "not planned". The community workaround is `npx mcp-remote <url>` as a stdio bridge. UNVERIFIED as an official statement, but treat Desktop as stdio-only.

## 3. MCP specification facts that shape the server

Sources: https://modelcontextprotocol.io/specification/2026-07-28 and https://modelcontextprotocol.io/specification/2025-11-25

- **Two eras**. 2025-11-25 and earlier: `initialize` handshake, optional `Mcp-Session-Id`, endpoint supports POST and GET (GET for a server-initiated SSE stream, or 405). 2026-07-28: stateless, no handshake, every request carries protocol version and client capabilities in `_meta`, mandatory `server/discover`, required `Mcp-Method` and `Mcp-Name` headers, `subscriptions/listen` replaces GET and resource subscriptions, no SSE resumability. The spec's compatibility matrix says a dual-era server picks behaviour per request: modern `_meta` means stateless, an `initialize` request means legacy.
- **Security requirements for local HTTP servers** (both eras): MUST validate the `Origin` header and return 403 when present and invalid (DNS rebinding defence); SHOULD bind only to 127.0.0.1; SHOULD authenticate all connections. Security best practices for local servers: use stdio to limit access to the launching client, or if HTTP, require an authorization token or use a Unix domain socket with restricted access.
- **Authorization** is optional; the defined scheme is OAuth 2.1, but a static bearer token on loopback is consistent with the best-practices text even though it is not "MCP authorization" as specified. Stdio servers should take credentials from the environment, not implement OAuth.
- **stdio hygiene**: nothing but MCP messages on stdout, logging to stderr, exit promptly when stdin closes, client restarts the server if it exits unexpectedly.
- Custom transports over Unix domain sockets or TCP SHOULD reuse stdio framing (newline-delimited JSON-RPC).

## 4. App Sandbox rules for a local listener

Sources: Apple entitlement docs, TN3179, Apple Developer Forums posts by Apple DTS staff.

- **Entitlements**: `com.apple.security.network.server` allows listening; `com.apple.security.network.client` allows outgoing connections, explicitly including "a server process running on the same machine". Both are static, no runtime prompt. https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.network.server
- Swift idiom note: an entitlement is a key baked into the app's code signature; Xcode stores them in a `.entitlements` file and the sandbox enforces them at run time.
- **Loopback needs the server entitlement**: Apple documents no loopback exception, and a forum thread describes a sandboxed pair failing until both entitlements were added. UNVERIFIED as an explicit Apple sentence, but treat as required.
- **Local Network privacy (macOS 15+)** does not apply: loopback is not a broadcast-capable interface, listening never triggers it, and Terminal-launched processes are exempt anyway. https://developer.apple.com/documentation/technotes/tn3179-understanding-local-network-privacy
- **Application Firewall**: if the user has it on, macOS may show "accept incoming network connections?" for a listening app unless "automatically allow downloaded signed software" is on (the default) and the app is Developer ID signed with a stable designated requirement. Ad-hoc signed debug builds get prompted every build. UNVERIFIED whether a loopback-only listener is exempt.
- **Binding**: `NWListener` with `NWParameters.requiredLocalEndpoint = .hostPort(host: "127.0.0.1", port: ...)` or `requiredInterfaceType = .loopback`; use port 0 for an ephemeral port and read `listener.port` once `.ready`, or pick a stable port and fall back on conflict.

## 5. The stdio shim pattern

- **Apple does it**: Xcode 26 ships `mcpbridge`, a plain non-sandboxed stdio binary that finds the running Xcode by PID, launches Xcode if needed, and talks to it over XPC using private RunningBoard endpoint injection gated by a private entitlement. So the pattern is validated, but the exact plumbing is not available to third parties.
- **Can Claude Code exec a binary inside Fenro.app?** Yes, it is an ordinary path; every Mach-O in the bundle must be signed with hardened runtime for notarization. The shim must not use the `com.apple.security.inherit` entitlement (that is only for children spawned by the sandboxed app); leave it unsandboxed or give it its own sandbox with `network.client`. https://developer.apple.com/documentation/xcode/embedding-a-command-line-tool-in-a-sandboxed-app
- **App Translocation caveat**: a quarantined app launched from Downloads runs from a random temporary path. A config pointing at `Fenro.app/Contents/MacOS/fenro-mcp` written while translocated will break; write the config only after the app lives in `/Applications` or resolve the path at write time and explain the requirement.
- **How the shim reaches the app**, three options:
  1. **Loopback HTTP**: the shim is a tiny `mcp-remote`-style proxy that forwards stdio JSON-RPC to the app's Streamable HTTP endpoint. Simplest; reuses the same server for both client types.
  2. **Unix domain socket in an App Group container**: Apple's App Groups doc explicitly sanctions Unix sockets for sandboxed-to-non-sandboxed IPC, path must be in the group container (`~/Library/Group Containers/TEAMID.group/`). Watch the 104-byte `sun_path` limit in `NWListener`; BSD sockets can go to 253 with care. macOS 15 protects group containers with SIP; a Team-ID-prefixed group ID avoids prompts for Developer ID apps. UNVERIFIED: whether an unsigned non-entitled process can open a socket file there without a prompt. https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.application-groups
  3. **XPC directly to the app**: **not possible**. A GUI app cannot register a named Mach service at run time (`XPC_CONNECTION_MACH_SERVICE_LISTENER` "may only be passed for services advertised in the process' launchd.plist"). It would require a launchd agent registered through `SMAppService`, which is a separate helper process, not the app.
- **Auto-launch**: the shim can start the app when it is not running (`open -b <bundle id>` or `NSWorkspace.openApplication`), wait for the port or socket file, then connect. This gives "agent runs `claude`, Fenro appears" for free.

## 6. Lifecycle when the app is not running

- A listener owned by the GUI process dies with it. Options to survive: an `SMAppService` launch agent (a separate helper, appears in System Settings Login Items, on-demand via `MachServices`), or keep the app running as a menu-bar accessory (`LSUIElement`). https://developer.apple.com/documentation/servicemanagement/smappservice
- For Fenro the store lives in the app, so a headless agent would need its own database access and duplicate the domain layer. The shim's auto-launch covers the practical case: Claude Code starts, the shim launches Fenro, the developer sees the app. Charting already agreed the server is alive only while the app runs.

## 7. Authentication for a loopback server

- The spec requires Origin validation and recommends a token. The Swift SDK ships `OriginValidator.localhost(port:)`; a bearer check is a handler-level `httpContext` header comparison.
- Pattern used by other local servers: a random per-install token generated on first launch, stored in the app container, and written into the client config header (`Authorization: Bearer ...`). The stdio shim reads the token from the same file so the user never sees it. Rotating it means rewriting the config, so keep a "reconnect Claude Code" action in the app.
- Do not put the token in the URL; the spec forbids access tokens in the query string.

## 8. Implications for the "MCP transport and lifecycle decision"

**Recommendation: both, layered.** Host a Streamable HTTP server on 127.0.0.1 inside the app (needs the `network.server` entitlement), and bundle a thin stdio shim `fenro-mcp` that proxies to it and launches the app if needed.

- HTTP is the one server implementation; it serves Claude Code directly and any other HTTP-capable client.
- The shim costs a few hundred lines, gives Claude Desktop and other stdio-only clients a path, gives auto-launch, and passes `CLAUDE_PROJECT_DIR` through as a header so the server can resolve the Project from the caller's working directory. That last point solves a real gap: over plain HTTP the server has no way to know the caller's cwd.
- Port: prefer a fixed default (pick one in the dynamic range and document it) with a fallback to an ephemeral port written to a file the shim reads. Fixed makes the HTTP config a stable one-liner; the file makes the shim robust.
- Auth: per-install random bearer token, Origin validation, loopback bind. All three are spec requirements or recommendations.
- Protocol: implement the legacy 2025-11-25 handshake first (that is what the Swift SDK speaks and what Claude Code falls back to) and make sure the 2026-07-28 probe gets a clean rejection. Plan for dual-era once the SDK catches up.
- SDK: start with the official Swift SDK behind a small `NWListener` HTTP front end; it is thin, Swift 6 clean, and its `Server` actor is easy to test with `InMemoryTransport`. Keep SwiftMCP in reserve if the official SDK stays dormant.

**Conditions that flip the recommendation:**

- If the official SDK ships a listener or the 2026-07-28 revision, drop the custom HTTP front end.
- If the team decides Claude Desktop is not a target and cwd-based Project resolution is dropped, the shim can be deferred and plain HTTP suffices (the original charting lean).
- If Claude Code stops supporting the legacy handshake before the Swift SDK supports the new revision, switch to SwiftMCP or implement `server/discover` and the stateless request shape by hand; the surface is small.
- If the server must run with the app closed, revisit with an `SMAppService` agent, which changes the architecture (the agent would need its own store access).
