# HTTP Routing in Envoy

## Overview

The **router filter** (`envoy.filters.http.router`) is responsible for implementing **HTTP forwarding** in Envoy. It is essential for nearly all HTTP proxy scenarios. The **router filter** examines the routing table and determines whether to forward or redirect requests.

Routing in Envoy is primarily determined by the **incoming request headers** (e.g., `host` or `authority`) and how they map to an **upstream cluster** via **virtual hosts** and **routing rules**.

This section will explore how Envoy's HTTP routing works, covering **route configurations, virtual hosts, priorities, and matching rules**.

---

## Route Configuration

The **route configuration** (`route_config`) contains the routing table. While the router filter is the primary consumer of this table, other filters can also access it if they need to make routing-related decisions.

Each **route configuration** consists of multiple **virtual hosts**. Each virtual host has:

- **A logical name**
- **A set of domains** that it applies to
- **A set of routes** that define request matching criteria and corresponding actions

Envoy also supports **priority routing** at the route level. Each priority level has its own **connection pool** and **circuit-breaking** settings. The supported priorities are:

- **DEFAULT** (default priority if not specified)
- **HIGH** (higher priority handling)

### Example: Route Configuration

```yaml
route_config:
  name: my_route_config  # Name used for stats, not relevant for routing
  virtual_hosts:
  - name: bar_vhost
    domains: ["bar.io"]
    routes:
      - match:
          prefix: "/"
        route:
          priority: HIGH
          cluster: bar_io
  - name: foo_vhost
    domains: ["foo.io"]
    routes:
      - match:
          prefix: "/"
        route:
          cluster: foo_io
      - match:
          prefix: "/api"
        route:
          cluster: foo_io_api
```

In this example:
- Requests with `Host: bar.io` are matched to the **bar_vhost**.
- Requests with `Host: foo.io` are matched to the **foo_vhost**.
- If the request path starts with `/api`, it is routed to the **foo_io_api** cluster.
- If the request path starts with `/`, it is routed to **foo_io**.

---

## Route Matching Order

When an HTTP request arrives, Envoy follows these steps to determine routing:

1. **Virtual Host Matching:**
   - The `host` or `authority` header is matched against the `domains` field in each virtual host.
   - Example: `foo_vhost` is selected if `host` is `foo.io`.

2. **Route Matching:**
   - Once a virtual host is matched, Envoy checks the `routes` list.
   - It selects the **first matching route** based on the request path.

3. **Virtual Cluster Matching (if configured):**
   - If **virtual clusters** (`virtual_clusters`) exist, Envoy checks them for a match.
   - Virtual clusters allow **regex-based** route matching and custom statistics collection.

### Example: Matching Order Issue

```yaml
route_config:
  virtual_hosts:
  - name: hello_vhost
    domains: ["hello.io"]
    routes:
      - match:
          prefix: "/api"
        route:
          cluster: hello_io_api
      - match:
          prefix: "/api/v1"
        route:
          cluster: hello_io_api_v1
```

#### Expected Behavior:
- The request `curl hello.io/api/v1` might be expected to match `/api/v1`.
- However, **Envoy evaluates the routes in order**, so it matches `/api` **first**.
- The request is routed to `hello_io_api`, **not** `hello_io_api_v1`.

#### Solution:
- Swap the order of routes.
- Use a different matching rule (e.g., **exact match** instead of prefix matching).

### Corrected Order:

```yaml
route_config:
  virtual_hosts:
  - name: hello_vhost
    domains: ["hello.io"]
    routes:
      - match:
          prefix: "/api/v1"
        route:
          cluster: hello_io_api_v1
      - match:
          prefix: "/api"
        route:
          cluster: hello_io_api
```

Now, `curl hello.io/api/v1` will correctly match `hello_io_api_v1` before falling back to `/api`.

---

## Conclusion
- The **router filter** is responsible for HTTP request forwarding.
- Requests are matched to **virtual hosts**, which contain domains and routing rules.
- Route matching happens **in order**, and the **first match wins**.
- **Priority routing** allows different circuit-breaking and connection pool settings.
- **Virtual clusters** provide an advanced way to match routes and track custom statistics.

By understanding these principles, you can **fine-tune Envoy's routing behavior** for your applications.

