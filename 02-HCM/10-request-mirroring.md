# Request Mirroring in Envoy

## Overview
Request mirroring (also known as **traffic shadowing**) is a powerful technique in Envoy that allows duplicating live traffic to a secondary cluster without affecting the primary request flow. This is useful for **testing, debugging, and performance analysis** of new services before routing real traffic to them.

Mirrored requests are **fire-and-forget**, meaning Envoy does not wait for a response from the shadow cluster before sending the original request's response to the client.

---

## How Request Mirroring Works

- **Primary Traffic:** The request is routed to its intended upstream cluster.
- **Shadow Traffic:** A copy of the request is sent to a **mirrored cluster**, allowing developers to observe how the mirrored service handles real traffic.
- **No Impact on Production:** Since the mirrored traffic does not impact the original request flow, it's safe to use for load testing and debugging.
- **Host Header Modification:** Envoy automatically appends `-shadow` to the authority (host) header of the mirrored request.

---

## Configuration: Enabling Request Mirroring

Request mirroring is configured **at the route level** using the `request_mirror_policies` field.

### Example: Mirroring 100% of Requests

```yaml
route_config:
  name: my_route
  virtual_hosts:
  - name: httpbin
    domains: ["*"]
    routes:
    - match:
        prefix: "/"
      route:
        cluster: httpbin
        request_mirror_policies:
          cluster: mirror_httpbin
          runtime_fraction:
            default_value:
              numerator: 100
```

### Explanation:
- The **primary request** is sent to the `httpbin` cluster.
- The **mirrored request** is sent to `mirror_httpbin`.
- **100% of requests** are mirrored (`numerator: 100`).
- The mirrored request's `Host` header will be modified to include `-shadow`.

---

## Controlling the Mirroring Percentage

To avoid overwhelming the shadow cluster, we can adjust the fraction of requests that get mirrored.

### Example: Mirroring Only 10% of Requests

```yaml
route_config:
  name: my_route
  virtual_hosts:
  - name: httpbin
    domains: ["*"]
    routes:
    - match:
        prefix: "/"
      route:
        cluster: httpbin
        request_mirror_policies:
          cluster: mirror_httpbin
          runtime_fraction:
            default_value:
              numerator: 10  # 10% of traffic
```

### Explanation:
- **Only 10% of requests** will be mirrored to `mirror_httpbin`.
- This helps reduce the impact on the shadow cluster while still allowing observation of live traffic.

---

## Key Considerations for Request Mirroring

| **Consideration**       | **Description** |
|-------------------------|----------------|
| **Mirrored Requests Are Fire-and-Forget** | The response from the shadow cluster is ignored, ensuring zero impact on production traffic. |
| **Use Only for Idempotent Requests** | Mirroring non-idempotent requests (e.g., `POST`, `DELETE`) may cause unintended side effects. |
| **Monitor Shadow Cluster Performance** | Ensure the shadow cluster can handle the additional traffic load. |
| **Header Modification** | The host header gets modified (`-shadow` appended). Adjust application logic if necessary. |

---

## Use Cases for Request Mirroring

1. **Testing New Services** - Validate how a new service handles live production traffic before full rollout.
2. **Performance Benchmarking** - Compare response times and error rates between the primary and shadow services.
3. **Canary Deployments** - Assess stability and correctness of a new deployment without affecting real users.
4. **Debugging and Observability** - Use monitoring tools to analyze real-time traffic without impacting production.

---

## Summary
Request mirroring in Envoy provides a safe and effective way to test services under real traffic conditions. By carefully **controlling the mirroring percentage**, monitoring **shadow cluster health**, and ensuring **idempotency**, teams can **gain valuable insights** without affecting live users.

By leveraging request mirroring, organizations can **accelerate development, validate deployments, and improve service reliability** while keeping production traffic intact.

