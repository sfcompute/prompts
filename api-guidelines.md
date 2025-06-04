# API Guidelines

## **URL Design**

### **Resources**

- Use plural nouns: `/orders`, `/clusters`, `/api_keys`
- Use snake_case for multi-word resources
- Limit nesting to 2 levels: `/orders/{id}/nodes/{id}`
- Use nested paths only when sub-resource cannot exist without parent

### **Endpoints**

```
GET    /orders           # List orders
GET    /orders/{id}      # Get orders
POST   /orders           # Create orders
PUT    /orders/{id}      # Replace orders
PATCH  /orders/{id}      # Update orders
DELETE /orders/{id}      # Delete orders
```

### **Query Parameters**

- Use snake_case: `?created_after=1234567890&limit=50`
- Sorting: `?sort=created_at` or `?sort=-created_at`
  - Follows the JSON:API spec for sorting with minus prefix for descending order
- Filtering: `?status=active&verified=true`
  - Follows the OpenAPI 3.0 spec for query parameter filtering
- Arrays/lists: `?status=active&status=expired&status=pending`
  - Follows the OpenAPI 3.0 spec for array query parameters using repeated keys

## **Versioning**

### **Deprecation**

- Announce deprecation 6+ months in advance
- Include deprecation warnings in response headers:

    ```
    Deprecation: true
    Sunset: Sat, 31 Dec 2024 23:59:59 GMT
    ```

## **HTTP Status Codes**

### **Success**

- `200` OK - GET, PUT, PATCH success
- `201` Created - POST success
- `204` No Content - DELETE success

### **Client Errors**

- `400` Bad Request - Validation errors
- `401` Unauthorized - Authentication required
- `403` Forbidden - Insufficient permissions
- `404` Not Found - Resource doesn't exist
- `409` Conflict - Duplicate resource or idempotency conflict
- `422` Unprocessable Entity - Semantic errors
- `429` Too Many Requests - Rate limit exceeded

### **Server Errors**

- `500` Internal Server Error
- `502` Bad Gateway
- `503` Service Unavailable
- `504` Gateway Timeout

## **Error Format**

```json
{
  "error": {
    "type": "invalid_request_error",
    "message": "One or more fields are invalid",
    "details": [
      {
        "field": "email",
        "code": "invalid_format",
        "message": "Email must be a valid email address"
      }
    ],
    "request_id": "req_1234567890"
  }
}
```

### **Error Types**

- `api_error` - Server-side errors
- `invalid_request_error` - Client-side errors
- `idempotency_error` - Idempotency key reused with different parameters

## **Authentication**

### **API Keys**

```
Authorization: Bearer sk_live_1234567890
```

- Prefix by environment: `sk_live_`, `sk_test_`
- Return `401` for auth failures

## **Rate Limiting**

### **Headers (all responses)**

These are the draft 9 headers for the ietf spec.

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640995200
```

### **Rate Limit Key Best Practices**

### **Key Selection Strategy**

**Authenticated Endpoints**

```rust
// Primary key (recommended default)let rate_limit_key = format!("account:{}:endpoint:{}", account_id, request_path);

// Alternative: API Key ID with path (if multiple keys per account)let rate_limit_key = format!("api_key:{}:path:{}", api_key_id, request_path);
```

**Public Endpoints (No Auth Required)**

We use JA3 fingerprints and IP for the key. We don't rely on only IP address because many network setups (like corporate offices), have many devices that will share the same IP address. We don't want to block an entire office building if there's only 1 bad individual actor.

```rust
// Primary key (recommended default)let rate_limit_key = format!("ja3:{}:ip:{}:endpoint:{}", ja3_hash, client_ip, endpoint_name);
```

### **Monitoring and Analytics**

Rate limits are tracked and alerted on.

**Key Metrics to Track**

- **Key Distribution** - Track request volume by key type (account-based, IP-based, etc.)
- **Hit Rates** - Monitor percentage of requests hitting rate limits by key type
- **Top Consumers** - Identify accounts/IPs with highest usage patterns
- **Limit Violations** - Track frequency and patterns of rate limit breaches

**Alerting Thresholds**

- Alert when hit rates exceed 10% for any key type
- Monitor for unusual spikes in key cardinality
- Track accounts consistently hitting limits
- Alert on rate limiting system performance degradation

## **Idempotency**

Endpoints should accept an idempotency key for actions that shouldn't be repeated based on the same input.

### **Implementation**

- Accept `Idempotency-Key` header on POST requests
- Use UUIDs or high-entropy random strings
- Store results for 24 hours
- Return `409` if key reused with different parameters

### **Usage**

```
POST /orders
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

## **Data Formats**

### **Conventions**

- **Timestamps:** Unix epoch seconds (UTC): `1640995200`
  - Should be suffixed with `_at` e.g. `created_at` instead of `created`
  - This rule applies to URL params, body fields, response fields, everything!
- **Currency:** Cents as integers: `2500` (for $25.00)
- **IDs:** Prefixed strings: `user_1234567890`, `cluster_abc123`
- **Booleans:** `true`/`false` only
- **Field keys:** Should be snake_case

### **Resource Structure**

```json
{
  "id": "user_1234567890",
  "object": "user",
  "email": "user@example.com",
  "created_at": 1640995200,
  "updated_at": 1640995200
}
```

## **Pagination**

### **List Endpoints**

Use cursor-based pagination with mutually exclusive parameters:

- `limit` (1-100, default 10)
- `starting_after` - Object ID cursor for next page
- `ending_before` - Object ID cursor for previous page

### **Response Format**

```json
{
  "object": "list",
  "url": "/v1/orders",
  "has_more": true,
  "data": [...]
}
```

## **Request/Response Headers**

### **Required Headers**

**Request:**

```
Content-Type: application/json
Accept: application/json
```

**Response:**

`X-SFC-Request-Id` is the same value as `request_id`.

```
Content-Type: application/json
X-SFC-Request-Id: req_1234567890
```

## **Breaking Changes**

### **Never Break**

- Remove fields from responses
- Change field types or formats
- Add required parameters
- Change HTTP status codes
- Modify error response structure

### **Safe Changes**

- Add optional fields to responses
- Add optional parameters to requests
- Add new endpoints
- Deprecate (but continue supporting) fields

## **Monitoring**

### **Required Metrics Per Endpoint**

Track these metrics for every API endpoint to ensure comprehensive observability:

### **Request Metrics**

- **api_requests_total** - Total requests by method, endpoint, and status code
- **api_requests_per_second** - Request rate by method and endpoint
- **api_concurrent_requests** - Active concurrent requests by method and endpoint
- **api_request_duration_p50** - 50th percentile response time in milliseconds
- **api_request_duration_p95** - 95th percentile response time in milliseconds
- **api_request_duration_p99** - 99th percentile response time in milliseconds
- **api_request_duration_max** - Maximum response time in milliseconds
- **api_request_size_bytes** - Request payload size in bytes
- **api_response_size_bytes** - Response payload size in bytes

### **Error and Status Metrics**

- **api_error_rate** - Error rate by method, endpoint, and error type
- **api_status_code_total** - Total requests by method, endpoint, and status code
- **api_2xx_total** - Successful requests by method and endpoint
- **api_4xx_total** - Client error requests by method and endpoint
- **api_5xx_total** - Server error requests by method and endpoint
- **api_validation_errors_total** - Validation errors by method, endpoint, and field
- **api_authentication_failures_total** - Authentication failures by method and endpoint
- **api_authorization_failures_total** - Authorization failures by method and endpoint

### **Rate Limiting Metrics**

- **api_rate_limit_hits_total** - Rate limit violations by method, endpoint, and limit type
- **api_rate_limit_remaining** - Remaining requests for account and limit type
- **api_rate_limit_resets_total** - Rate limit resets by account and limit type
- **api_rate_limit_keys_active** - Active rate limit keys by key type
- **api_rate_limit_key_cardinality** - Total unique rate limit keys by key type

### **Performance Metrics**

- **api_external_call_duration_ms** - External API call duration by method, endpoint, and service
- **api_external_call_failures_total** - External API call failures by method, endpoint, and service

### **Logging Guidelines**

### **Required Log Fields**

**Every API request must log these fields:**

- **timestamp** - ISO 8601 formatted timestamp
- **level** - Log level (info, warn, error, debug)
- **request_id** - Unique request identifier
- **method** - HTTP method (GET, POST, etc.)
- **endpoint** - Normalized endpoint path
- **path** - Full request path including parameters
- **status_code** - HTTP response status code
- **duration_ms** - Request duration in milliseconds
- **request_size_bytes** - Request payload size
- **response_size_bytes** - Response payload size
- **user_agent** - Client user agent string
- **account_id** - Account identifier (if authenticated)
- **api_key_id** - API key identifier (if authenticated)

**Error requests must include additional fields:**

- **error_type** - Category of error (validation_error, auth_error, etc.)
- **error_code** - Specific error code
- **error_message** - Human-readable error message
- **error_field** - Field that caused validation error (if applicable)
- **stack_trace** - Stack trace (only for 5xx errors)
- **user_context** - Additional user context (account tier, feature flags)

### **Log Levels**

**INFO** - Log successful requests (200-299 status codes)

**WARN** - Log client errors (400-499 status codes)

**ERROR** - Log server errors (500-599 status codes)

**DEBUG** - Log detailed tracing information (development only)

**Rate Limit Violations** - Log when rate limits are exceeded:

- Rate limit key that was exceeded
- Current limit and count
- Time window for the limit
- Endpoint that triggered the violation

### **Alerting Configuration**

### **Critical Alerts (Immediate Response)**

- **5xx Error Rate** - Alert when > 1% over 5 minute window
- **Response Time P99** - Alert when > 5000ms over 5 minute window
- **Rate Limit Hit Rate** - Alert when > 25% over 10 minute window

### **Warning Alerts (Monitor Closely)**

- **4xx Error Rate** - Alert when > 10% over 15 minute window
- **Response Time P95** - Alert when > 2000ms over 10 minute window
- **Database Connection Pool** - Alert when > 80% utilization over 5 minute window

### **Key Metrics to Track**

- **Key Distribution** - Track request volume by key type (account-based, IP-based, etc.)
- **Hit Rates** - Monitor percentage of requests hitting rate limits by key type
- **Top Consumers** - Identify accounts/IPs with highest usage patterns
- **Limit Violations** - Track frequency and patterns of rate limit breaches

### **Alerting Thresholds**

- Alert when hit rates exceed 10% for any key type
- Monitor for unusual spikes in key cardinality
- Track accounts consistently hitting limits
- Alert on rate limiting system performance degradation
