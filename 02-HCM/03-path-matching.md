# Path Matching in Envoy

Envoy provides various rules to match incoming requests based on different path-matching techniques. These rules help in defining precise routing strategies for handling requests efficiently.

## Path Matching Rules

The table below describes different path-matching strategies supported by Envoy:

| Rule Name       | Description |
|----------------|-------------|
| **prefix**     | Matches if the beginning of the path header matches the prefix. For example, the prefix `/api` will match paths `/api` and `/api/v1`, but not `/`. |
| **path**       | Matches only if the entire path header matches exactly (excluding query parameters). For example, `/api` will match `/api`, but not `/api/v1` or `/`. |
| **safe_regex** | Matches the path against the specified regular expression. For example, `^/products/\d+$` will match `/products/123` and `/products/321` but not `/products/hello` or `/api/products/123`. |
| **connect_matcher** | Matches only `CONNECT` requests (currently in Alpha). |

📌 *By default, `prefix` and `path` matching are case-sensitive. To make them case-insensitive, set `case_sensitive` to `false`. This setting does not apply to `safe_regex`.*

---

## Headers Matching

Headers matching allows requests to be routed based on specific headers present in the request. The router checks the request headers against the specified headers in the route configuration.

### Range Match

The `range_match` rule checks if a request header value falls within a specified integer range. The range is defined using an inclusive start and an exclusive end (`[start, end)`).

```yaml
- match:
    prefix: "/"
    headers:
    - name: minor_version
      range_match:
        start: 1
        end: 11
```
This example matches requests where the `minor_version` header has a value between `1` and `10`.

### Present Match

The `present_match` rule checks for the presence of a specific header in the request.

```yaml
- match:
    prefix: "/"
    headers:
    - name: debug
      present_match: true
```
If the `debug` header is present, the match evaluates as `true`, regardless of the header value.

### String Match

Envoy supports various string-matching rules for headers:

```yaml
- match:
    prefix: "/"
    headers:
    - name: regex_match
      string_match:
        safe_regex_match:
          google_re2: {}
          regex: "^v\\d+$"
    - name: exact_match
      string_match:
        exact: "hello"
    - name: prefix_match
      string_match:
        prefix: "api"
    - name: suffix_match
      string_match:
        suffix: "_1"
    - name: contains_match
      string_match:
        contains: "debug"
```

- `safe_regex_match`: Matches using a regular expression.
- `exact_match`: The header must contain the exact specified value.
- `prefix_match`: The header value must start with the given prefix.
- `suffix_match`: The header value must end with the given suffix.
- `contains_match`: The header value must contain the specified substring.

### Invert Match

The `invert_match` rule negates the match condition.

```yaml
- match:
    prefix: "/"
    headers:
    - name: version
      range_match:
        start: 1
        end: 6
      invert_match: true
```
Here, the match evaluates to `true` if the `version` header is **not** in the range `1-5`.

```yaml
- match:
    prefix: "/"
    headers:
    - name: env
      contains_match: "test"
      invert_match: true
```
This rule ensures that the `env` header does not contain the string `test`.

---

## Query Parameters Matching

Query parameters can also be used for request matching. Envoy checks the query string in the `path` header and compares it against the provided parameters.

```yaml
- match:
    prefix: "/"
    query_parameters:
    - name: env
      present_match: true
```
This rule checks if the request contains a query parameter named `env`, without validating its value.

### String Matching for Query Parameters

| Rule Name | Description |
|-----------|-------------|
| **exact** | Must exactly match the query parameter value. |
| **prefix** | The parameter value must start with the specified string. |
| **suffix** | The parameter value must end with the specified string. |
| **safe_regex** | The parameter value must match the specified regular expression. |
| **contains** | The parameter value must contain a specified substring. |

Example:

```yaml
- match:
    prefix: "/"
    query_parameters:
    - name: env
      string_match:
        prefix: "env_"
        ignore_case: true
```
Here, requests where `env` starts with `env_` (case-insensitive) will match.

---

## gRPC and TLS Matchers

### gRPC Matcher

Envoy can match only gRPC requests using the `grpc` matcher.

```yaml
- match:
    prefix: "/"
    grpc: {}
```
This rule matches if the request is a gRPC request.

### TLS Context Matcher

The `tls_context` matcher validates whether a request presents a valid TLS certificate.

```yaml
- match:
    prefix: "/"
    tls_context:
      presented: true
      validated: true
```
This rule evaluates to `true` if the request includes a certificate that has been presented and validated.

