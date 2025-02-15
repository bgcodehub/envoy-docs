# Rate Limiting Statistics

## Overview
Envoy emits various statistics when rate limiting is enabled, whether using **global** or **local** rate-limiting mechanisms. These metrics provide insights into the number of requests processed, rate-limited requests, errors, and other rate-limiting behavior.

To configure the statistics prefix, the `stat_prefix` field must be set when configuring rate limit filters.

### Metric Naming Convention
Metric names follow a structured format:
- **Local Rate Limiting Metrics**:
  - `<stat_prefix>.http_local_rate_limit.<metric_name>`
- **Global Rate Limiting Metrics**:
  - `cluster.<route_target_cluster>.ratelimit.<metric_name>`

This naming convention allows easy identification of local versus global rate-limiting metrics in monitoring and observability tools.

---
## Rate Limiting Metrics
The table below details the key rate-limiting metrics emitted by Envoy:

| **Rate Limiter** | **Metric Name**             | **Description**  |
|----------------|---------------------------|------------------|
| **Local**     | `enabled`                   | Total number of requests for which the rate limiter was called. |
| **Local/Global** | `ok`                     | Total number of under-limit responses from the token bucket (requests that were allowed). |
| **Local**     | `rate_limited`              | Total responses without an available token (but not necessarily enforced). |
| **Local**     | `enforced`                  | Total number of rate-limited requests (e.g., HTTP 429 is returned). |
| **Global**    | `over_limit`                | Total over-limit responses from the external rate limit service. |
| **Global**    | `error`                     | Total errors encountered while contacting the external rate limit service. |
| **Global**    | `failure_mode_allowed`      | Total requests that encountered errors but were allowed due to the `failure_mode_deny` setting being false. |

These metrics are critical in monitoring rate limit efficiency, debugging rate-limiting behavior, and ensuring fair usage policies are effectively enforced.

---
## Practical Usage
### Example: Monitoring with Prometheus
When using Prometheus for monitoring Envoy's rate-limiting behavior, you can query metrics as follows:

#### Check total rate-limited requests:
```
rate(envoy_http_local_rate_limit_enforced[5m])
```

#### Check requests that exceeded global limits:
```
rate(cluster.my_service.ratelimit_over_limit[5m])
```

#### Check rate limiter errors:
```
rate(cluster.my_service.ratelimit_error[5m])
```

These queries help observe trends, detect anomalies, and fine-tune rate-limiting policies to ensure smooth operations.

---
## Conclusion
Envoy provides detailed statistics to monitor and analyze rate limiting at both **local** and **global** levels. By leveraging these metrics, operators can optimize request handling, ensure fair resource allocation, and detect potential issues related to rate-limiting configurations.

