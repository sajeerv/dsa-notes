# WebSockets: System Design Concept Note

WebSockets provide a persistent, bidirectional connection between a client and server. Use them when both sides need to exchange messages with low latency, without opening a new HTTP request for every update.

## 1. Mental model and lifecycle

```text
Browser ── HTTP Upgrade request ──> WebSocket endpoint
Browser <── 101 Switching Protocols ── Server
Browser <════ persistent, bidirectional connection ════> Server
```

1. The client starts with an HTTP handshake requesting an upgrade to WebSocket.
2. The server validates the handshake and may authenticate/authorize the connection.
3. The server responds `101 Switching Protocols` when it accepts.
4. Both sides can now send WebSocket messages at any time over the same TCP connection.
5. Either side can start a WebSocket close handshake.

`ws://` is unencrypted; production browser applications should use `wss://`, which protects the connection with TLS. The WebSocket protocol defines framing and a connection lifecycle, but **does not define your application's message schema, user identity, rooms, persistence, or delivery guarantees**. Those are application responsibilities.

Classic WebSockets use an HTTP/1.1 Upgrade handshake; an extension also defines bootstrapping WebSockets over HTTP/2. The familiar handshake uses HTTP so existing web infrastructure can route and authorize the initial connection.

## 2. How it differs from other approaches

| Approach | Direction | Connection pattern | Good fit | Main tradeoff |
|---|---|---|---|---|
| Short polling | Client asks server repeatedly | Many short HTTP requests | Simple, low update frequency | Wasted requests and update delay between polls |
| Long polling | Client makes a request held until data/timeout, then reconnects | Repeated HTTP requests | Broad compatibility; fallback transport | Repeated request overhead and more complicated connection handling |
| Server-Sent Events (SSE) | Server → client | Persistent HTTP event stream; client sends separately via HTTP | Notifications, feeds, progress updates | One-way stream; separate channel needed for client messages |
| WebSocket | Client ↔ server | Persistent, full-duplex connection | Chat, presence, collaboration, interactive dashboards | Stateful connections consume resources; reconnect and fanout need design |
| WebRTC data channel | Peer ↔ peer (typically) | Peer connection established with signaling | Low-latency peer data, voice/video adjunct data | Signaling, NAT traversal, and peer connectivity add complexity |

Choose the simplest transport that fits the communication pattern. A dashboard with server-only updates may be well served by SSE. A chat app where clients send and receive continuously is a natural WebSocket use case. WebSocket does not automatically make delivery durable or peer-to-peer.

## 3. Message protocol and connection behavior

WebSocket messages can carry text or binary payloads. A practical application protocol usually defines:

- `type` or `action` to identify the message, such as `chat.send` or `presence.update`.
- `requestId` / `correlationId` to match responses to requests.
- `eventId` or sequence/cursor where clients need deduplication or resume.
- A versioning policy and error format.
- Limits for message size, rate, and subscriptions.

Example:

```json
{"type":"chat.send","requestId":"r-17","roomId":"room-4","text":"Hi"}
```

Do not assume that an open socket means a user is still online or able to receive every message. Networks drop connections, devices sleep, and proxies close idle connections. Use ping/pong heartbeats or application heartbeats, detect dead connections, and reconnect with exponential backoff plus jitter. If messages must not be lost during disconnects, store them durably and let clients resume from a cursor; the socket by itself is not a queue.

## 4. Spring Boot: endpoint, port, and URL

Start Spring Boot normally on its configured HTTP port (for example, `8080`) and register a WebSocket handler at a path such as `/ws`. A browser client connects to that endpoint with a WebSocket client:

```javascript
const socket = new WebSocket("ws://localhost:8080/ws");
```

You usually do not open `http://localhost:8080/ws` as a webpage. The WebSocket client begins with an HTTP handshake to that path and asks Spring to upgrade the connection. If accepted, that same connection remains open for messages in both directions. Spring does not open a new server-side socket for every message; the client initiates a connection, and the server accepts it.

Spring Boot does not generate a separate `wss://` URL. `wss://` is the secure form of the WebSocket URL and requires TLS on the connection path:

- If Spring Boot serves HTTPS directly on port `8443`, connect to `wss://localhost:8443/ws` (with a trusted certificate for real deployments).
- In production, TLS is commonly terminated at a reverse proxy or load balancer. Clients connect to a public URL such as `wss://api.example.com/ws`; the proxy forwards the upgraded connection to Spring Boot internally.

Keep the endpoint path and public host/port aligned with the server and proxy configuration. A `wss://` URL alone does not enable TLS; the certificate and TLS configuration must be in place.

## 5. Scaling WebSockets

### Connection placement

A load balancer routes the initial handshake to an application node. Once upgraded, that TCP connection stays attached to that node until it closes. The node therefore holds local connection state (socket, user/session, subscriptions, buffers).

Scale horizontally by running multiple WebSocket nodes behind a WebSocket-capable load balancer or gateway. Plan for **concurrent connections**, per-connection memory/file descriptors, connection setup rate, message rate, outbound fanout, and network egress. CPU alone is not a useful capacity measure. Configure idle timeouts and ping behavior consistently across clients, proxies, and load balancers.

### Cross-node message delivery

If a user on node A sends a message to a user connected to node B, node A needs a route to node B. Common patterns:

```text
Client → WebSocket node → broker/pub-sub → WebSocket node(s) → recipients
```

- Keep each live socket in the memory of the node that owns it.
- Publish cross-node events through a shared backplane such as Redis Pub/Sub, NATS, or a messaging system appropriate to the product.
- Maintain room/user-to-node membership in a shared or partitioned directory if targeted delivery requires it.
- Use durable storage or a durable stream for messages that must survive disconnections; ephemeral pub/sub is for live fanout and may lose events while a consumer is offline.

**Sticky sessions:** an established WebSocket TCP connection is already bound to its node, so it does not move between requests. Affinity can still matter for reconnects, a multi-request negotiation flow, and libraries that fall back to HTTP long polling. Socket.IO documentation specifically requires affinity for its multi-server long-polling setup. Verify requirements for the chosen framework and transport.

### Reconnects, delivery, and draining

- Reconnect with backoff and jitter to avoid a reconnect storm after a node/region outage.
- Expect reconnect to land on a different node. Re-authenticate and restore subscriptions.
- For resumable streams, give events sequence IDs and retain enough history to replay after the last acknowledged cursor.
- Decide whether sends are at-most-once, at-least-once with deduplication, or best-effort. Do not claim exactly-once delivery from WebSocket itself.
- During deploys, stop accepting new connections, notify/close existing connections, allow clients to reconnect, and avoid terminating all nodes simultaneously.
- Apply backpressure: bound outbound queues, cap message sizes, drop/coalesce replaceable updates, or disconnect persistently slow consumers. The classic browser WebSocket API does not provide application-level backpressure automatically.

### Choosing a backplane

| Need | Common choice |
|---|---|
| Ephemeral cross-node broadcast | Redis Pub/Sub, NATS |
| Durable event log / replay / stream processing | Kafka or another durable log, with a separate delivery/fanout layer as needed |
| Managed WebSocket connections and routing | Cloud gateway/service such as Amazon API Gateway WebSocket APIs or Azure SignalR Service |
| Small deployment | In-process connection map, understanding it is single-node only |

Choose based on ordering, replay, fanout, throughput, operations, and failure semantics—not just familiarity. A shared broker is not automatically a durable message history.

## 6. Authentication and authorization

WebSocket has no built-in user authentication. Authenticate at connection establishment and authorize each action/message afterward.

### Browser authentication patterns

The native browser `WebSocket` constructor accepts a URL and optional subprotocols, not an arbitrary headers object. This affects how browser clients send bearer tokens.

1. **Secure session cookie:** browser sends the cookie during the handshake. Use `wss://`, secure `HttpOnly` cookies, appropriate `SameSite`, and validate the `Origin` header against an explicit allowlist. Cookie authentication without Origin checks can enable Cross-Site WebSocket Hijacking.
2. **Authenticate in the first message:** establish WSS, send a short-lived access token immediately, and allow no protected action or data until it is validated. Apply a short auth timeout and close unauthenticated sockets. Never put long-lived bearer tokens in query strings because URLs can appear in logs.
3. **Short-lived one-time connection ticket:** obtain a narrowly scoped, short-lived ticket over authenticated HTTPS, then present it during WebSocket connection setup. Ensure it is single-use or tightly scoped and avoid logging it.

Native/non-browser clients can often set an `Authorization` header during handshake, depending on their WebSocket library. Support this when appropriate, but keep authorization rules consistent across client types.

### Authorization after connection

- Check permissions for every command and room/channel subscription; authentication is not blanket authorization.
- Bind the authenticated identity to the connection server-side; do not trust a client-supplied user ID.
- Handle token/session expiry, revocation, and logout by revalidating or closing associated connections.
- Validate message schemas and enforce payload size, rate limits, and per-user connection limits.
- Log connection/auth/security metadata, but never log tokens, session IDs, or sensitive message bodies.

## 7. Common implementation stacks

| Language/platform | Common options | Notes |
|---|---|---|
| JavaScript/TypeScript, Node.js | Native browser `WebSocket`; `ws`; Socket.IO | `ws` is a low-level RFC 6455 implementation. Socket.IO adds events, rooms, acknowledgements, reconnection behavior, and adapters; it is a higher-level protocol and is not wire-compatible with a plain WebSocket client. |
| Java / Spring | Spring WebSocket, Spring Messaging with STOMP | STOMP can add destinations and publish/subscribe conventions on top of WebSocket. Use raw WebSocket when a custom lightweight protocol is preferred. |
| .NET | ASP.NET Core SignalR | Higher-level hubs, groups, reconnect patterns, and transport fallback; supports WebSockets, SSE, and long polling. |
| Python | FastAPI/Starlette WebSocket endpoints, Django Channels, `websockets` | Async server support; multi-process deployments still need a channel layer/backplane for cross-worker messaging. |
| Go | `coder/websocket`, `gobwas/ws` | Choose a library based on API style, context/cancellation, performance needs, and maintenance status. |
| Managed cloud | Amazon API Gateway WebSocket APIs, Azure SignalR Service | Reduces connection-server operations; application still owns auth, routing/business logic, persistence, and delivery semantics. |

Pick a higher-level framework when rooms, RPC-like methods, fallback transports, or connection management save meaningful work. Pick a lower-level library when you need direct protocol control and are ready to implement those features yourself.

## 8. Common interview questions

**Why not use HTTP polling?** Polling is simpler and may be enough at low update rates; WebSocket reduces repeated request overhead and supports immediate traffic both ways.

**Does WebSocket guarantee delivery?** No. It provides an ordered byte/message stream while connected over TCP. Application acknowledgements, persistence, replay, retries, and deduplication are separate design choices.

**How do you broadcast across several servers?** Keep sockets local to their owner node and use a backplane/broker to route events to nodes with interested clients. Keep durable history separately if offline clients must catch up.

**How do you handle a reconnect?** Re-authenticate, restore subscriptions, and resume from a client cursor if the product needs missed events. Use jittered backoff.

**How do you prevent unauthorized cross-site connections?** Use WSS, validate the browser `Origin` against an allowlist, authenticate the handshake or first message, and authorize every action.

## References

- [RFC 6455: The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455.html)
- [RFC 8441: Bootstrapping WebSockets with HTTP/2](https://www.rfc-editor.org/rfc/rfc8441.html)
- [MDN: WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)
- [MDN: Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [MDN: WebRTC data channels](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Using_data_channels)
- [OWASP WebSocket Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/WebSocket_Security_Cheat_Sheet.html)
- [NGINX: WebSocket proxying](https://nginx.org/en/docs/http/websocket.html)
- [Socket.IO: Adapters and multi-server scaling](https://socket.io/docs/v4/adapter/)
- [Spring Framework: WebSockets](https://docs.spring.io/spring/reference/web/webflux-websocket.html)
- [Microsoft Learn: ASP.NET Core SignalR](https://learn.microsoft.com/en-us/aspnet/core/signalr/introduction)
- [FastAPI: WebSockets](https://fastapi.tiangolo.com/advanced/websockets/)
- [Amazon API Gateway: WebSocket routes](https://docs.aws.amazon.com/apigateway/latest/developerguide/websocket-api-develop-routes.html)
