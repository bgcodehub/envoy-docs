# Retries in Envoy

## Overview
Envoy allows retrying requests based on configurable **retry policies** at both the **virtual host** and **route level**. This feature is useful for handling transient failures, improving availability, and ensuring resilience in distributed systems. 

- **Virtual Host Retry Policy**: Applies to all routes under a virtual host.
- **Route-Level Retry Policy**: Overrides the virtual host policy for specific routes.

In addition to configuration-based policies, retry behavior can be controlled dynamically using request headers such as `x-envoy-retry-on`.

---

## Configurable Retry Settings
Envoy allows configuring various aspects of retries:

### Maximum Number of Retries
- **Defines how many times Envoy retries a request** before failing.
- Uses an **exponential backoff algorithm** for retry intervals by default.
- All retries must occur within the total request timeout (`request_timeout`).
- Default retries: **1**.

### Example Configuration:
```yaml
route_config:
  virtual_hosts:
  - name: httpbin
    domains: ["*"]
    routes:
    - match:
        path: /status/500
      route:
        cluster: httpbin
        retry_policy:
          retry_on: "5xx"
          num_retries: 5
```
- Retries **5 times** when receiving a **500 response code**.

#### Log Output Example (if retries are exhausted):
```
[2021-07-26T18:43:29.515Z] "GET /status/500 HTTP/1.1" 500 URX 0 0 269 269 "-" "curl/7.64.0" "1ae9ffe2-21f2-43f7-ab80-79be4a95d6d4" "localhost:10000" "127.0.0.1:5000"
```
**`500 URX`** indicates that the upstream response was 500, and retries were exhausted.

---

## Retry Conditions
Envoy allows retrying requests based on different conditions. The `retry_on` field specifies when a retry should be attempted.

| **Retry Condition** | **Description** |
|---------------------|----------------|
| `5xx` | Retries on 5xx response codes and upstream failures. |
| `gateway-error` | Retries on 502, 503, or 504 errors. |
| `reset` | Retries if upstream does not respond. |
| `connect-failure` | Retries on connection failures (e.g., timeouts). |
| `envoy-ratelimited` | Retries if `x-envoy-ratelimited` header is present. |
| `retriable-4xx` | Retries retriable 4xx responses (currently only 409). |
| `refused-stream` | Retries if upstream resets the stream. |
| `retriable-status-codes` | Retries for user-defined status codes in `x-envoy-retriable-status-codes`. |
| `retriable-headers` | Retries based on matching headers in `x-envoy-retriable-header-names`. |

---

## Host Selection for Retries
By default, Envoy retries using **random host selection**. However, this behavior can be customized using **host selection retry plugins**.

### Example: Avoiding Previously Attempted Hosts
```yaml
route_config:
  virtual_hosts:
  - name: httpbin
    domains: ["*"]
    routes:
    - match:
        path: /status/500
      route:
        cluster: httpbin
        retry_policy:
          retry_host_predicate:
          - name: envoy.retry_host_predicates.previous_hosts
          host_selection_retry_max_attempts: 5
```
- This ensures that retries **do not** go to hosts that have already failed.

---

## Request Hedging
**Request hedging** sends multiple requests **simultaneously** and uses the first successful response.

### Example: Enabling Request Hedging on Timeout
```yaml
route_config:
  virtual_hosts:
  - name: httpbin
    domains: ["*"]
    hedge_policy:
      hedge_on_per_try_timeout: true
    routes:
    - match:
        path: /status/500
      route:
        cluster: httpbin
```
- If an initial request **times out**, Envoy issues a retry request **without canceling the original request**.
- The **first valid response** is returned to the client.

---

## Summary

| **Feature** | **Description** |
|------------|----------------|
| **Maximum Retries** | Defines how many times Envoy retries a request. |
| **Retry Conditions** | Specifies when a retry should occur (5xx, timeouts, etc.). |
| **Host Selection** | Avoids selecting failed hosts for retries. |
| **Request Hedging** | Sends multiple requests in parallel and picks the fastest response. |

By fine-tuning retry policies and host selection strategies, Envoy ensures better resiliency and fault tolerance in modern distributed systems.

