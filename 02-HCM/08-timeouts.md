# Timeouts in Envoy

## Overview
Envoy supports multiple configurable timeouts that apply depending on the proxy's use case. These timeouts can be set at different levels, including the **HTTP Connection Manager (HCM)** level and **route-level**, allowing flexibility in request handling. Some higher-level timeouts can be overridden at lower levels, providing granular control over request and response handling.

---

## Configurable Timeouts

### 1. **Request Timeout**
The `request_timeout` specifies how long Envoy waits for the **entire request to be received**.

- The timeout is triggered when the request starts and deactivated once the last byte of the request is sent to the upstream service or when the response is initiated.
- Default: **Disabled** (set to `0` if not explicitly provided).

#### Example Configuration:
```yaml
filters:
- name: envoy.filters.network.http_connection_manager
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
    request_timeout: 120s  # Set request timeout to 120 seconds
```

---

### 2. **Idle Timeout**
The `idle_timeout` determines when a **downstream or upstream connection** is closed if no active streams are present.

- Default: **1 hour**.
- This timeout is defined in the **common_http_protocol_options** of the **HCM configuration**.
- Can also be set for **upstream clusters**.

#### Example Configuration:
```yaml
filters:
- name: envoy.filters.network.http_connection_manager
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
    common_http_protocol_options:
      idle_timeout: 600s  # 10 minutes
```

For **upstream clusters**, the configuration is:
```yaml
clusters:
- name: upstream_cluster
  connect_timeout: 5s
  common_http_protocol_options:
    idle_timeout: 300s  # 5 minutes
```

---

### 3. **Request Headers Timeout**
The `request_headers_timeout` defines the **maximum time Envoy waits for request headers to be received**.

- The timer starts when the first byte of headers is received and stops when the last byte is received.
- Default: **Disabled** (set to `0` if not provided).

#### Example Configuration:
```yaml
filters:
- name: envoy.filters.network.http_connection_manager
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
    request_headers_timeout: 5s  # Set header timeout to 5 seconds
```

---

### 4. **Stream Idle Timeout**
The `stream_idle_timeout` is used for **streaming responses** where the standard request timeout is not applicable.

- Useful for **gRPC and long-lived connections**.
- If a response takes too long without activity, the connection is closed.

#### Example Configuration:
```yaml
filters:
- name: envoy.filters.network.http_connection_manager
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
    stream_idle_timeout: 1800s  # 30 minutes
```

---

### 5. **Drain Timeout**
The `drain_timeout` controls how long Envoy waits before closing existing connections when **shutting down or draining connections**.

#### Example Configuration:
```yaml
filters:
- name: envoy.filters.network.http_connection_manager
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
    drain_timeout: 20s  # Set drain timeout to 20 seconds
```

---

### 6. **Delayed Close Timeout**
The `delayed_close_timeout` allows the proxy to delay closing a connection to **allow clients to read response data**.

#### Example Configuration:
```yaml
filters:
- name: envoy.filters.network.http_connection_manager
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
    delayed_close_timeout: 5s  # Delay close by 5 seconds
```

---

### 7. **Route-Level Timeout**
The **route timeout** (`timeout`) defines how long Envoy waits for an **upstream service to respond**.

- Default: **15 seconds**.
- Overrides `request_timeout` at the HCM level.
- Needs to be **disabled for streaming responses**.

#### Example Configuration:
```yaml
route_config:
  virtual_hosts:
  - name: service_host
    domains: ["example.com"]
    routes:
    - match:
        prefix: "/api"
      route:
        cluster: backend_service
        timeout: 10s  # Set route-specific timeout to 10 seconds
```

---

### 8. **Per-Try Timeout**
The `per_try_timeout` applies when **retries are enabled**, limiting the time for each retry attempt.

- **Shorter than `timeout`** (total request timeout).

#### Example Configuration:
```yaml
route_config:
  virtual_hosts:
  - name: service_host
    domains: ["example.com"]
    routes:
    - match:
        prefix: "/api"
      route:
        cluster: backend_service
        timeout: 20s  # Overall request timeout
        retry_policy:
          per_try_timeout: 5s  # Max time per retry attempt
          num_retries: 3  # Allow up to 3 retries
```

---

## Summary of Timeouts
| Timeout | Description | Default Value |
|---------|-------------|---------------|
| **Request Timeout** | Time to receive the full request | **Disabled** (0) |
| **Idle Timeout** | Time before terminating idle connections | **1 hour** |
| **Request Headers Timeout** | Time to receive request headers | **Disabled** (0) |
| **Stream Idle Timeout** | Time before closing long-lived streams | **N/A** |
| **Drain Timeout** | Grace period before closing draining connections | **N/A** |
| **Delayed Close Timeout** | Extra time before closing connection | **N/A** |
| **Route-Level Timeout** | Max time waiting for an upstream response | **15 seconds** |
| **Per-Try Timeout** | Max time per retry attempt | **N/A** |

By properly configuring these timeouts, we can **enhance request handling efficiency, prevent resource exhaustion, and ensure smooth service operation**. Carefully tuning timeout values per service and workload **optimizes performance and resilience** in an Envoy-proxied environment.

