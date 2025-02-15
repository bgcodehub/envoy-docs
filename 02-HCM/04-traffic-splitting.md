# Traffic Splitting in Envoy

## Overview
Envoy supports **traffic splitting** to different routes within the same virtual host, enabling gradual rollouts, canary releases, A/B testing, and multivariate testing scenarios. Traffic can be split between two or more upstream clusters using **runtime percentages** or **weighted clusters**.

---
## Approach 1: Traffic Splitting Using Runtime Percentages
Runtime percentages allow for dynamic traffic shifting without changing the static configuration, making them ideal for **progressive delivery** and **canary releases**.

### Example Configuration:
```yaml
route_config:
  virtual_hosts:
  - name: hello_vhost
    domains: ["hello.io"]
    routes:
      - match:
          prefix: "/"
          runtime_fraction:
            default_value:
              numerator: 90
              denominator: HUNDRED
        route:
          cluster: hello_v1
      - match:
          prefix: "/"
        route:
          cluster: hello_v2
```
### Explanation:
- **90% of traffic** is routed to `hello_v1`.
- **10% of traffic** is routed to `hello_v2`.
- Envoy generates a random number between `[0, denominator)`, and if it is **less than the numerator**, it matches the first route (`hello_v1`). Otherwise, it continues to `hello_v2`.
- Once the numerator is set to `0`, **all traffic** will go to `hello_v2`.

### Dynamic Control via Runtime Keys
We can store the **numerator** in a runtime key, allowing adjustments without modifying the configuration:
```yaml
route_config:
  virtual_hosts:
  - name: hello_vhost
    domains: ["hello.io"]
    routes:
      - match:
          prefix: "/"
          runtime_fraction:
            default_value:
              numerator: 0
              denominator: HUNDRED
            runtime_key: routing.hello_io
        route:
          cluster: hello_v1
      - match:
          prefix: "/"
        route:
          cluster: hello_v2
...
layered_runtime:
  layers:
  - name: static_layer
    static_layer:
      routing.hello_io: 90
```
- The value is now controlled dynamically under `routing.hello_io`, allowing runtime updates via file or Runtime Discovery Service (RTDS).

---
## Approach 2: Traffic Splitting Using Weighted Clusters
Weighted clusters provide a more **scalable** way to split traffic across multiple services. Instead of defining separate routes, we configure **weighted clusters** within a **single route**.

### Example Configuration:
```yaml
route_config:
  virtual_hosts:
  - name: hello_vhost
    domains: ["hello.io"]
    routes:
      - match:
          prefix: "/"
        route:
          weighted_clusters:
            clusters:
              - name: hello_v1
                weight: 90
              - name: hello_v2
                weight: 10
```
### Explanation:
- **90% of traffic** goes to `hello_v1`, and **10%** to `hello_v2`.
- Unlike the **runtime fraction approach**, **all traffic is evaluated within a single route**.
- More flexible than the runtime fraction approach, especially for **A/B testing and multi-cluster traffic splitting**.

### Dynamically Controlled Weights
By adding a `runtime_key_prefix`, we can **dynamically adjust traffic weights** at runtime:
```yaml
route_config:
  virtual_hosts:
  - name: hello_vhost
    domains: ["hello.io"]
    routes:
      - match:
          prefix: "/"
        route:
          weighted_clusters:
            runtime_key_prefix: routing.hello_io
            clusters:
              - name: hello_v1
                weight: 90
              - name: hello_v2
                weight: 10
...
layered_runtime:
  layers:
  - name: static_layer
    static_layer:
      routing.hello_io.hello_v1: 90
      routing.hello_io.hello_v2: 10
```
- The router reads `routing.hello_io.hello_v1` and `routing.hello_io.hello_v2` values dynamically from **runtime keys**.
- If the keys don’t exist, **default weights** are used.

### Controlling Total Weights
By default, **weights must sum to 100**. However, we can override this by setting a `total_weight` field:
```yaml
route_config:
  virtual_hosts:
  - name: hello_vhost
    domains: ["hello.io"]
    routes:
      - match:
          prefix: "/"
        route:
          weighted_clusters:
            runtime_key_prefix: routing.hello_io
            total_weight: 15
            clusters:
              - name: hello_v1
                weight: 5
              - name: hello_v2
                weight: 5
              - name: hello_v3
                weight: 5
```
### Explanation:
- The `total_weight` is set to **15**, meaning Envoy will scale weights accordingly.
- The values are dynamically controlled using the runtime key prefix.
- Helps when **scaling traffic across multiple clusters** dynamically.

---
## Summary
| **Method** | **Best For** | **Advantages** | **Challenges** |
|------------|-------------|----------------|---------------|
| **Runtime Percentages** | Canary Releases, Progressive Delivery | Dynamic updates, No need for redeployments | Limited to two clusters, requires manual weight balancing |
| **Weighted Clusters** | A/B Testing, Multi-cluster Splitting | Scales well to multiple clusters, Single-route evaluation | Requires `total_weight` adjustment if scaling dynamically |

By using **runtime percentages** for gradual rollouts and **weighted clusters** for scalable traffic distribution, we can efficiently manage and control traffic flow within Envoy. These approaches empower teams to perform **progressive rollouts**, **A/B testing**, and **multi-version deployments** with precision.