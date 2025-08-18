# Achilleus Security Implementation

## SSRF Protection (Critical)

### NetworkGuard Class
```php
class NetworkGuard {
    public static function assertPublicHttpUrl(string $url): void;
}
```

**Blocked Patterns**:
- Localhost: 127.0.0.1, ::1, localhost
- Private Networks: 10.x.x.x, 192.168.x.x, 172.16-31.x.x
- Cloud Metadata: 169.254.169.254
- Internal Services: 0.0.0.0
- Non-HTTP protocols: ftp://, file://, etc.

## Input Validation

### Form Requests
Always use Laravel Form Requests for validation:

```php
class StoreDomainRequest extends FormRequest {
    public function rules(): array {
        return [
            'url' => [
                'required',
                'url',
                'starts_with:https://',
                new PublicUrl(),
                new UniqueDomainForUser(auth()->id())
            ]
        ];
    }
}
```

### Custom Validation Rules
- `PublicUrl`: Prevents private IP addresses
- `UniqueDomainForUser`: Domain uniqueness per user
- `MaxDomainsForUser`: Enforces 10 domain limit

## Authentication & Authorization

### Domain Ownership
```php
class DomainPolicy {
    public function view(User $user, Domain $domain): bool {
        return $user->id === $domain->user_id;
    }
}
```

### Route Protection
All domain operations require authorization:
```php
$this->authorize('view', $domain);
```

## Rate Limiting

### API Rate Limits
```php
// In RouteServiceProvider
RateLimiter::for('scans', function (Request $request) {
    return Limit::perMinute(10)->by($request->user()->id);
});
```

### Scanner Rate Limits
Each scanner implements host-based rate limiting to be respectful to target servers.

## Data Security

### Sensitive Data Handling
- Never log URLs, API keys, or personal data
- Use encrypted casting for sensitive database fields
- Sanitize error messages in production

### Database Security
- UUID primary keys (no sequential IDs)
- Foreign key constraints with CASCADE deletes
- CHECK constraints for data integrity
- Proper indexes for performance

## HTTP Client Security

### Secure Configuration
```php
Http::timeout(30)
    ->withUserAgent('Achilleus Security Scanner/1.0')
    ->withOptions([
        'verify' => true,                 // Verify SSL certificates
        'allow_redirects' => [
            'max' => 3,                   // Limit redirects
            'strict' => true,
            'referer' => false,
            'protocols' => ['https']      // HTTPS only
        ]
    ])
```

## Error Handling Security

### Production Error Messages
- Never expose internal paths or stack traces
- Generic error messages for users
- Detailed logging for developers
- Specific security incident logging

### Exception Sanitization
```php
return response()->json([
    'message' => 'Failed to start scan',
    'error' => app()->environment('production') 
        ? 'Internal server error' 
        : $e->getMessage()
], 500);
```

## Queue Security

### Job Validation
All queue jobs validate input data:
- Domain ownership verification
- URL safety checks
- User permission validation

### Retry Limits
Jobs have retry limits to prevent infinite loops:
```php
public $tries = 3;
public $timeout = 60;
public $backoff = [10, 30, 60];
```

## Monitoring & Alerting

### Security Events
Log critical security events:
- SSRF attempt blocked
- Rate limit exceeded
- Invalid domain access
- Payment failures

### Audit Trail
Track sensitive operations:
- Domain additions/deletions
- Subscription changes
- Data exports
- Account deletions