# Achilleus API Reference

## Authentication
All API endpoints require authentication via Laravel's session-based auth (Inertia.js).

## Domain Management

### List Domains
```
GET /domains
```

**Response**:
```json
{
  "domains": [
    {
      "id": "uuid",
      "url": "example.com",
      "display_name": "example.com",
      "last_scan_at": "2024-01-15T10:30:00Z",
      "last_scan_score": 85,
      "is_active": true
    }
  ]
}
```

### Add Domain
```
POST /domains
Content-Type: application/json

{
  "url": "https://example.com",
  "email_mode": "expected"
}
```

### Delete Domain
```
DELETE /domains/{domain}
```

## Scan Management

### Trigger Scan
```
POST /domains/{domain}/scan
```

**Response**:
```json
{
  "message": "Scan started successfully",
  "scan_id": "uuid"
}
```

### Get Scan Results
```
GET /scans/{scan}
```

**Response**:
```json
{
  "id": "uuid",
  "status": "completed",
  "total_score": 85,
  "grade": "B+",
  "modules": [
    {
      "module": "ssl_tls",
      "score": 90,
      "status": "ok",
      "raw": {
        "certificate_valid": true,
        "protocol_version": "TLSv1.3",
        "expires_at": "2025-01-15"
      }
    }
  ]
}
```

## Reports

### Generate Report
```
POST /reports
Content-Type: application/json

{
  "scan_id": "uuid",
  "format": "pdf"
}
```

### Download Report
```
GET /reports/{report}/download
```

## Billing

### Subscription Status
```
GET /billing/subscription
```

**Response**:
```json
{
  "status": "active",
  "plan": "solo",
  "current_period_end": "2024-02-15T00:00:00Z",
  "cancel_at_period_end": false
}
```

### Create Subscription
```
POST /billing/subscribe
Content-Type: application/json

{
  "payment_method_id": "pm_stripe_id"
}
```

### Cancel Subscription
```
POST /billing/cancel
```

## Webhook Endpoints

### Stripe Webhooks
```
POST /webhooks/stripe
```

Handles:
- `subscription.created`
- `subscription.updated` 
- `subscription.deleted`
- `payment_failed`

## Error Responses

### Validation Errors
```json
{
  "message": "The given data was invalid.",
  "errors": {
    "url": ["The url format is invalid."]
  }
}
```

### Rate Limit Errors
```json
{
  "message": "Too many requests. Please wait and try again."
}
```

### Server Errors
```json
{
  "message": "Internal server error"
}
```

## Rate Limits
- Scan endpoints: 10 requests per minute per user
- Domain management: 30 requests per minute per user
- General API: 60 requests per minute per user

## Response Format
All successful responses follow this format:
- 200 OK: Successful request with data
- 201 Created: Resource created successfully  
- 204 No Content: Successful request, no response body
- 400 Bad Request: Validation or request errors
- 401 Unauthorized: Authentication required
- 403 Forbidden: Insufficient permissions
- 404 Not Found: Resource not found
- 429 Too Many Requests: Rate limit exceeded
- 500 Internal Server Error: Server error