# Header Manipulation in Envoy

## Overview
Envoy supports **header manipulation** at multiple levels within the HTTP Connection Manager (HCM). Headers can be added, removed, or modified at the **weighted cluster, route, virtual host, and global levels**.

While most headers can be altered, certain immutable headers such as pseudo-headers (e.g., `:scheme`) and `host` cannot be directly modified. However, headers like `:path` and `:authority` can be **indirectly modified** via configurations like `prefix_rewrite`, `regex_rewrite`, and `host_rewrite`.

## Header Modification Order
Headers are applied in the following order:
1. **Weighted cluster-level headers**
2. **Route-level headers**
3. **Virtual host-level headers**
4. **Global-level headers**

Headers configured at a higher level **can overwrite** headers set at lower levels.

## Header Modification Fields
Each level supports the following fields for header manipulation:
- **`response_headers_to_add`**: Adds headers to the response.
- **`response_headers_to_remove`**: Removes headers from the response.
- **`request_headers_to_add`**: Adds headers to the request.
- **`request_headers_to_remove`**: Removes headers from the request.

### Example Configuration
```yaml
route_config:
  response_headers_to_add:
    - header:
        key: "header_1"
        value: "some_value"
      append: false
  response_headers_to_remove: "header_we_dont_need"
  virtual_hosts:
  - name: hello_vhost
    request_headers_to_add:
      - header:
          key: "v_host_header"
          value: "from_v_host"
    domains: ["hello.io"]
    routes:
      - match:
          prefix: "/"
        route:
          cluster: hello
        response_headers_to_add:
          - header:
              key: "route_header"
              value: "%DOWNSTREAM_REMOTE_ADDRESS%"
      - match:
          prefix: "/api"
        route:
          cluster: hello_api
        response_headers_to_add:
          - header:
              key: "api_route_header"
              value: "api-value"
          - header:
              key: "header_1"
              value: "this_will_be_overwritten"
```

### Dynamic Variables in Headers
Headers can use **dynamic variables** enclosed within `%` symbols. Some commonly used variables:
- `%DOWNSTREAM_REMOTE_ADDRESS%` - The remote client IP address.
- `%UPSTREAM_REMOTE_ADDRESS%` - The upstream server's IP address.
- `%START_TIME%` - The request start timestamp.
- `%RESPONSE_FLAGS%` - Response-related metadata.

## Standard Headers Set by Envoy
Envoy automatically sets certain headers when processing requests.

### Request Headers (Decoding Phase)
```plaintext
:authority: localhost:10000
:path: /
:method: GET
:scheme: http
user-agent: curl/7.64.0
accept: */*
x-forwarded-proto: http
x-request-id: 14f0ac76-128d-4954-ad76-823c3544197e
x-envoy-expected-rq-timeout-ms: 15000
```

### Response Headers (Encoding Phase)
```plaintext
:status: 200
x-powered-by: Express
content-type: text/html; charset=utf-8
content-length: 563
etag: W/"233-b+4UpNDbOtHFiEpLMsDEDK7iTeI"
date: Fri, 16 Jul 2021 21:59:52 GMT
x-envoy-upstream-service-time: 2
server: envoy
```

### Explanation of Key Headers
| **Header** | **Description** |
|------------|----------------|
| `:scheme` | Set and forwarded upstream. For HTTP/1, derived from `x-forwarded-proto`. |
| `user-agent` | Set by the client; can be modified with `add_user_agent`. |
| `x-forwarded-proto` | Identifies the original client connection protocol (HTTP or HTTPS). |
| `x-request-id` | Uniquely identifies a request for tracing and logging. |
| `x-envoy-expected-rq-timeout-ms` | Timeout expected for the request (default: 15s). |
| `x-envoy-upstream-service-time` | Time (ms) the upstream service took to process the request. |
| `server` | Defaults to `envoy`; can be modified via `server_name` setting. |

## Header Sanitization
Header sanitization helps enforce security by modifying or stripping sensitive headers.

### Headers Subject to Sanitization
| **Header** | **Purpose** |
|------------|------------|
| `x-envoy-downstream-service-cluster` | Identifies the downstream caller’s cluster. |
| `x-envoy-expected-rq-timeout-ms` | Indicates expected request timeout. |
| `x-envoy-external-address` | Stores the client’s external IP. |
| `x-forwarded-for` | Tracks the proxy chain IPs. |
| `x-forwarded-client-cert` | Contains client certificate information. |
| `x-envoy-retry-on` | Specifies the retry policy. |

## X-Forwarded-For (XFF) and Request Origin Determination
The **X-Forwarded-For (XFF)** header tracks the chain of proxies a request passed through. Envoy evaluates this to determine if a request is **internal or external**.

By default, Envoy **does not append** the client’s IP to `XFF`. This behavior is controlled using:
- `use_remote_address: true` → Uses the real client IP as the source.
- `skip_xff_append: false` → Appends the client IP to `XFF`.

### Example Configuration
```yaml
...
- filters:
  - name: envoy.filters.network.http_connection_manager
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
      use_remote_address: true
      skip_xff_append: false
```

### Internal Request Example (Local Machine)
```plaintext
x-forwarded-for: 10.128.0.17
x-envoy-internal: true
```
- `x-forwarded-for`: Internal IP address of the caller.
- `x-envoy-internal`: Indicates the request is internal.

### External Request Example (Outside Network)
```plaintext
x-forwarded-for: 50.35.69.235
x-envoy-external-address: 50.35.69.235
```
- `x-forwarded-for`: Contains the client’s external IP.
- `x-envoy-external-address`: Identifies the original client address.

## Conclusion
Envoy provides extensive **header manipulation capabilities** to help manage request forwarding, response transformations, security enforcement, and logging enhancements. Understanding these features allows fine-grained control over traffic flow and request processing.

