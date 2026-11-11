# Proxy Servers and Reverse Proxies

## 1. What is a proxy server?

A **proxy** is an intermediary that receives network traffic and forwards it onward. The names *forward proxy* and *reverse proxy* describe which side the proxy represents.

### Forward proxy

A forward proxy sits on the **client side**. Clients send outbound requests through it, and it connects to destination servers on their behalf.

```text
Client(s) → Forward proxy → Internet / destination servers
```

Typical uses include outbound access control in a company, egress IP management, content filtering, privacy, and caching shared outbound resources. The destination generally sees the proxy as the network peer (although application headers can reveal more).

### Reverse proxy

A reverse proxy sits on the **server side**. Clients connect to the proxy, which selects and forwards each request to an origin or backend service.

```text
Browser / mobile app → Reverse proxy → Web app / API servers
```

Clients use the public hostname of the proxy and usually do not know which backend handled a request. A reverse proxy can protect and organize the server side, while a forward proxy controls or represents clients.

## 2. Reverse proxy request flow

1. DNS resolves the application hostname to the proxy, load balancer, or edge network.
2. The client connects to that public endpoint, typically using HTTPS.
3. The proxy applies configured policies, such as TLS handling, routing, filtering, or caching.
4. It forwards the request to a selected backend, often over a private network.
5. The backend response returns through the proxy to the client.

```text
                     ┌─ /api/* ─→ API service
Client ─→ Proxy ─────┼─ /app/* ─→ Web service
                     └─ /static/* → cache or object storage
```

## 3. What can a reverse proxy do?

| Feature | What it does | Typical reason to use it |
|---|---|---|
| Request routing | Routes by hostname, path, headers, or other request attributes | Put several services behind one public hostname |
| Load balancing | Distributes new requests/connections across backend instances | Use capacity and avoid overloading one instance |
| Health checks and failover | Stops selecting backends that fail configured checks | Avoid sending traffic to unavailable instances |
| TLS termination | Decrypts HTTPS at the proxy; may re-encrypt to backends | Centralize certificates and reduce backend TLS work |
| Caching and compression | Reuses eligible responses or compresses data | Reduce latency, bandwidth, and origin load |
| Security controls | Can integrate WAF rules, DDoS protection, request limits, and access policy | Filter or limit unwanted traffic before it reaches the app |
| Header and URL handling | Adds or rewrites headers, paths, and redirects | Pass routing/context information or support migrations |
| Observability | Records request metrics, access logs, and traces | See traffic and diagnose routing/backend failures |
| Protocol translation | Bridges supported protocols, such as HTTP/2 at the edge to HTTP/1.1 upstream | Let clients and backends use different supported protocols |

These are capabilities of products and configurations, not automatic properties of every proxy. For example, caching must be configured with correct cache keys and rules; personalized or sensitive responses should not be shared accidentally.

## 4. Proxy vs. load balancer, CDN, API gateway, and WAF

These terms overlap because one product can provide several roles.

| Term | Main job | Relationship to a reverse proxy |
|---|---|---|
| Reverse proxy | Receives traffic for backend servers and forwards it | Broad architectural role |
| Load balancer | Distributes traffic among targets and avoids unhealthy targets | Often a reverse proxy, especially at Layer 7; Layer 4 balancers can forward TCP/UDP flows without understanding HTTP paths |
| CDN | Serves content from distributed edge locations, often caching responses | Commonly acts as a globally distributed reverse proxy in front of origins |
| API gateway | Applies API-specific routing, identity, quotas, transformations, and lifecycle policies | Specialized reverse-proxy role for APIs |
| WAF | Inspects and blocks selected web requests based on security rules | Often integrated into a reverse proxy/CDN/load balancer, but is a security function rather than the routing role itself |
| Ingress controller / Gateway | Configures and implements traffic entry into a container platform | Usually programs reverse-proxy data-plane instances such as NGINX or Envoy |

**Layer 4 vs. Layer 7:** Layer 4 operates on connections and transport addresses (for example TCP or UDP). Layer 7 understands application protocols such as HTTP and can route by host, path, or headers. Use Layer 4 when you need generic high-throughput connection forwarding or protocol transparency; use Layer 7 when application-aware routing or HTTP policy is needed.

## 5. Common software and managed providers

### Self-managed proxy software

| Product | Common role and strengths | Considerations |
|---|---|---|
| NGINX / NGINX Plus | HTTP reverse proxy, web server, caching, TLS termination, and load balancing | Open-source NGINX and commercial NGINX Plus differ in available capabilities and support; check the edition and feature requirements |
| HAProxy / HAProxy Enterprise or ALOHA | High-performance Layer 4/7 proxy and load balancer; health checks and detailed traffic policies | Requires operating and managing the proxy fleet/configuration; commercial offerings add enterprise features/support |
| Envoy Proxy | L7 proxy for edge traffic and service-to-service communication; dynamic configuration, observability, advanced balancing, retries, and circuit-breaker capabilities | More concepts and control-plane/configuration complexity than a small static proxy setup |
| Apache HTTP Server (`mod_proxy`) | Reverse proxy integrated with the Apache web server ecosystem | Useful when already using Apache; evaluate operational and performance fit against dedicated proxies |

### Managed cloud and edge services

| Provider/product | Typical role and features |
|---|---|
| AWS Application Load Balancer (ALB) | Managed Layer 7 HTTP/HTTPS routing, target groups, TLS termination, and WebSocket support. AWS Network Load Balancer (NLB) is for Layer 4 connection/flow balancing. CloudFront is AWS's CDN/edge service. |
| Google Cloud Load Balancing | Managed application and network load balancers; global and regional options, health checks, autoscaling, WebSockets, and integrations with Cloud CDN and Cloud Armor. |
| Azure Application Gateway | Regional web traffic load balancer with URL/path routing, TLS termination, WAF integration, WebSockets, and HTTP/2 support. Azure Front Door provides a global edge entry/routing service. |
| Cloudflare | Proxied DNS can place Cloudflare's network between visitors and the origin; product options include CDN caching, WAF, DDoS protection, and load balancing/health-based steering. |
| Fastly | Edge CDN/reverse proxy and programmable edge services, with options including caching and edge WAF. |

The best fit depends on whether you want to operate proxy instances yourself, need global edge caching/security, need Layer 4 or Layer 7 behavior, and which cloud/network you already use. Product names and feature availability vary by region, tier, and configuration; verify the provider's current feature matrix before choosing.

## 6. Common deployment patterns

### One proxy in front of several services

```text
api.example.com ─┐
app.example.com ─┼→ Reverse proxy / load balancer → backend services
static.example.com┘
```

The proxy routes requests to the right service by hostname and/or path.

### CDN / edge plus regional load balancer

```text
User → CDN / edge proxy → regional load balancer → application instances
```

The edge layer can cache public content and filter traffic near users. The regional balancer routes uncached or dynamic requests to healthy backend instances.

### Kubernetes ingress

```text
Internet → cloud load balancer → ingress/gateway proxy → Kubernetes service → pods
```

The ingress/gateway configuration maps hostnames and paths to cluster services. A Kubernetes Ingress or Gateway object is configuration; a controller or gateway implementation performs the actual proxying.

## 7. TLS and client IP handling

### TLS choices

- **Terminate at proxy, plaintext upstream:** simpler, but acceptable only when the internal network and threat model allow it.
- **Terminate and re-encrypt:** proxy decrypts client TLS, applies HTTP policies, and opens a separate TLS connection to the backend.
- **TLS passthrough:** proxy forwards encrypted connections without inspecting HTTP content; useful when end-to-end TLS at the backend is required, but removes most Layer 7 routing and inspection options.

With TLS termination, the proxy is a trusted security boundary: it can see plaintext application traffic. Protect certificates and proxy-to-backend traffic appropriately.

### Forwarded headers

Proxies often add headers such as `Forwarded` or `X-Forwarded-For` to convey original client information. Configure the application to trust these headers **only from known proxy hops**. If the backend accepts arbitrary client-supplied forwarded headers, a client may spoof its IP or scheme. Restrict direct public access to the backend where possible, and configure the trusted-proxy chain carefully.

## 8. Reliability and security checklist

- Make the backend reachable only from intended proxy/network paths where feasible.
- Use health checks that reflect whether an instance can serve traffic, not merely whether its process accepts a TCP connection.
- Set connection, idle, request, and upstream timeouts intentionally; align them across proxy and application.
- Preserve or replace the `Host` header and forwarded headers according to an explicit policy.
- Set body/header size limits, rate limits, and connection limits appropriate to the application.
- Decide whether retries are safe. Retrying a non-idempotent `POST` can duplicate side effects unless the application has idempotency protection.
- Avoid caching private or user-specific responses unless cache keys and directives guarantee isolation.
- Monitor proxy health, upstream latency/errors, active connections, TLS expiry, saturation, and rejected requests.
- Plan config validation, staged rollout, and rollback; a bad central proxy change can affect all traffic.
- Use redundant proxy instances or a managed highly available service; a single proxy can become a bottleneck or single point of failure.

## 9. Interview summary

“A reverse proxy is the server-side entry point that receives client traffic and forwards it to backend services. It can combine routing, load balancing, health checks, TLS termination, caching, security policy, and observability. A forward proxy represents clients making outbound requests. For an internet-facing system, I’d put a highly available Layer 7 proxy or managed application load balancer in front of the services, use health checks and TLS, restrict direct backend access, and add a shared cache/CDN or WAF if the workload needs those features. I’d preserve client identity through trusted proxy headers, and I’d only retry requests when their side effects are safe.”

## References

- [NGINX: Reverse proxy and HTTP load balancing](https://nginx.org/en/docs/http/load_balancing.html)
- [HAProxy: Health checks](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/reliability/health-checks/)
- [Envoy: What is Envoy?](https://www.envoyproxy.io/docs/envoy/latest/intro/what_is_envoy.html)
- [AWS: Application Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html)
- [AWS: How Elastic Load Balancing works](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)
- [Google Cloud: Load balancing overview](https://docs.cloud.google.com/load-balancing/docs/load-balancing-overview)
- [Microsoft Learn: Azure Application Gateway features](https://learn.microsoft.com/en-us/azure/application-gateway/features)
- [Cloudflare: Proxy status](https://developers.cloudflare.com/dns/proxy-status/)
- [Cloudflare: Load balancing reference architecture](https://developers.cloudflare.com/reference-architecture/architectures/load-balancing/)
- [Fastly: What is a reverse proxy?](https://www.fastly.com/learning/cdn/what-is-a-reverse-proxy)
