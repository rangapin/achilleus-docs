

# Achilleus Testing Strategy

## Testing Philosophy

### Core Principles
- **Test-Driven Development (TDD)**: Write tests before implementation
- **Real Data Testing**: Use actual external services for integration tests
- **Security-First Testing**: Every feature must include security validation tests
- **Performance Benchmarks**: Critical paths must meet performance requirements

## Testing Stack

- **Unit Tests**: Pest PHP (Laravel's modern testing framework)
- **Feature Tests**: Laravel's built-in HTTP testing
- **E2E Tests**: Playwright for browser automation
- **Performance Tests**: Laravel's benchmark helpers
- **Security Tests**: Custom security assertion helpers

## Test Organization

```
tests/
├── Unit/
│   ├── Scanners/
│   ├── Services/
│   └── Support/
├── Feature/
│   ├── Api/
│   ├── Auth/
│   └── Billing/
├── E2E/
│   ├── UserFlows/
│   └── SecurityScans/
└── Fixtures/
    └── TestDomains.php
```

## What to Test vs Not Test

### ✅ ALWAYS Test
- Security validations (SSRF, input validation)
- Business logic (scoring algorithms, grade calculations)
- API endpoints and responses
- Authorization policies
- Database constraints and relationships
- Scanner retry logic and timeouts
- Rate limiting enforcement
- Trial period calculations
- Domain limit enforcement

### ❌ DON'T Test
- Laravel framework internals
- Third-party package functionality
- External service availability
- CSS styling and animations
- Database engine behavior
- Laravel Cloud infrastructure

## Unit Testing Patterns

### Scanner Testing

```php
test('ssl scanner detects expired certificates', function () {
    $scanner = app(SslTlsScanner::class);
    $result = $scanner->scan('https://expired.badssl.com');
    
    expect($result->score)->toBe(0);
    expect($result->status)->toBe('fail');
    expect($result->raw)->toHaveKey('certificate.expired', true);
});

test('ssl scanner awards bonus for TLS 1.3', function () {
    $scanner = app(SslTlsScanner::class);
    $result = $scanner->scan('https://github.com');
    
    expect($result->raw['protocol']['version'])->toBe('TLSv1.3');
    expect($result->score)->toBeGreaterThanOrEqual(90);
});
```

### SSRF Protection Testing

```php
test('network guard blocks localhost attempts', function () {
    expect(fn() => NetworkGuard::assertPublicHttpUrl('http://localhost'))
        ->toThrow(SecurityException::class, 'SSRF attempt blocked');
});

test('network guard blocks private IP ranges', function () {
    $privateIPs = [
        'http://192.168.1.1',
        'http://10.0.0.1',
        'http://172.16.0.1',
        'http://169.254.169.254', // AWS metadata
    ];
    
    foreach ($privateIPs as $ip) {
        expect(fn() => NetworkGuard::assertPublicHttpUrl($ip))
            ->toThrow(SecurityException::class);
    }
});
```

### Scoring Algorithm Testing

```php
test('scoring engine calculates weighted average correctly', function () {
    $scores = [
        'ssl_tls' => 80,
        'security_headers' => 90,
        'dns_email' => 70,
    ];
    
    $weights = [
        'ssl_tls' => 0.4,
        'security_headers' => 0.3,
        'dns_email' => 0.3,
    ];
    
    $engine = new ScoringEngine($weights);
    $result = $engine->calculate($scores);
    
    expect($result['total'])->toBe(80); // (80*0.4 + 90*0.3 + 70*0.3)
    expect($result['grade'])->toBe('B');
});

test('scoring engine redistributes weights when module fails', function () {
    $scores = [
        'ssl_tls' => 90,
        'security_headers' => null, // Failed
        'dns_email' => 80,
    ];
    
    $engine = new ScoringEngine();
    $result = $engine->calculate($scores);
    
    // Weight redistributed: SSL 57%, DNS 43%
    expect($result['total'])->toBe(86); // (90*0.57 + 80*0.43)
});
```

## Feature Testing Patterns

### API Testing

```php
test('domain creation enforces HTTPS-only policy', function () {
    $user = User::factory()->create();
    
    $response = $this->actingAs($user)
        ->postJson('/api/domains', [
            'url' => 'http://example.com',
            'email_mode' => 'expected',
        ]);
    
    $response->assertStatus(422)
        ->assertJsonValidationErrors(['url' => 'must use HTTPS']);
});

test('user cannot exceed domain limit', function () {
    $user = User::factory()->has(Domain::factory()->count(10))->create();
    
    $response = $this->actingAs($user)
        ->postJson('/api/domains', [
            'url' => 'https://example.com',
        ]);
    
    $response->assertStatus(403)
        ->assertJson(['message' => 'Domain limit reached']);
});
```

### Authorization Testing

```php
test('user cannot scan domains they do not own', function () {
    $owner = User::factory()->create();
    $otherUser = User::factory()->create();
    $domain = Domain::factory()->for($owner)->create();
    
    $response = $this->actingAs($otherUser)
        ->postJson("/api/domains/{$domain->id}/scan");
    
    $response->assertStatus(403);
});

test('trial expired users cannot initiate scans', function () {
    $user = User::factory()->create([
        'trial_ends_at' => now()->subDay(),
        'subscription_status' => 'expired',
    ]);
    
    $domain = Domain::factory()->for($user)->create();
    
    $response = $this->actingAs($user)
        ->postJson("/api/domains/{$domain->id}/scan");
    
    $response->assertStatus(403)
        ->assertJson(['message' => 'Active subscription required']);
});
```

## E2E Testing Patterns

### User Flow Testing

```typescript
test('complete user journey from signup to first scan', async ({ page }) => {
    // Registration
    await page.goto('/register');
    await page.fill('[name="email"]', 'test@example.com');
    await page.fill('[name="password"]', 'SecurePassword123!');
    await page.click('button[type="submit"]');
    
    // Verify trial banner
    await expect(page.locator('.trial-banner')).toContainText('14 days remaining');
    
    // Add domain
    await page.goto('/domains');
    await page.click('text=Add Domain');
    await page.fill('[name="url"]', 'https://github.com');
    await page.click('text=Add & Scan');
    
    // Wait for scan completion
    await page.waitForSelector('text=Scan completed', { timeout: 30000 });
    
    // Verify results
    await expect(page.locator('.security-score')).toBeVisible();
    await expect(page.locator('.grade-badge')).toContainText(/A\+?|A|B\+?/);
});

test('real-time scan updates via WebSocket', async ({ page }) => {
    const user = await createAuthenticatedUser();
    await page.goto('/dashboard');
    
    // Start scan
    await page.click('text=Scan All Domains');
    
    // Verify real-time status updates
    await expect(page.locator('.scan-status')).toContainText('Pending');
    await expect(page.locator('.scan-status')).toContainText('Running', { timeout: 5000 });
    await expect(page.locator('.scan-status')).toContainText('Completed', { timeout: 30000 });
});
```

## Performance Testing

### Response Time Requirements

```php
test('dashboard loads within 500ms', function () {
    $user = User::factory()
        ->has(Domain::factory()->count(10))
        ->has(Scan::factory()->count(50))
        ->create();
    
    $start = microtime(true);
    $response = $this->actingAs($user)->get('/dashboard');
    $duration = (microtime(true) - $start) * 1000;
    
    expect($duration)->toBeLessThan(500);
    $response->assertStatus(200);
});

test('scan completion within 30 seconds', function () {
    $domain = Domain::factory()->create(['url' => 'https://github.com']);
    
    $start = time();
    RunDomainScan::dispatchSync($domain);
    $duration = time() - $start;
    
    expect($duration)->toBeLessThan(30);
    expect($domain->fresh()->last_scan_score)->toBeGreaterThan(0);
});
```

## Security Testing Checklist

### Input Validation Tests
- [ ] URL validation (HTTPS-only)
- [ ] SSRF protection (localhost, private IPs)
- [ ] SQL injection prevention
- [ ] XSS protection in user inputs
- [ ] CSRF token validation

### Authentication Tests
- [ ] Trial period enforcement
- [ ] Subscription status checks
- [ ] OAuth flow validation
- [ ] Session security
- [ ] Password complexity

### Authorization Tests
- [ ] Domain ownership validation
- [ ] Resource access control
- [ ] Rate limiting enforcement
- [ ] API token scopes
- [ ] Admin vs user permissions

## Test Data Management

### Fixtures

```php
// tests/Fixtures/TestDomains.php
class TestDomains
{
    public static array $valid = [
        'https://github.com',
        'https://google.com',
        'https://stackoverflow.com',
    ];
    
    public static array $problematic = [
        'https://expired.badssl.com',        // Expired cert
        'https://wrong.host.badssl.com',      // Wrong hostname
        'https://self-signed.badssl.com',     // Self-signed
        'https://untrusted-root.badssl.com',  // Untrusted root
    ];
    
    public static array $invalid = [
        'http://example.com',      // Not HTTPS
        'https://localhost',        // Localhost
        'https://192.168.1.1',     // Private IP
        'ftp://example.com',       // Wrong protocol
    ];
}
```

### Database Seeding

```php
// database/seeders/TestDataSeeder.php
class TestDataSeeder extends Seeder
{
    public function run(): void
    {
        // Only run in testing/development
        if (app()->environment('production')) {
            return;
        }
        
        // Create test users with various states
        User::factory()->create([
            'email' => 'trial@test.com',
            'trial_ends_at' => now()->addDays(7),
        ]);
        
        User::factory()->create([
            'email' => 'expired@test.com',
            'trial_ends_at' => now()->subDay(),
        ]);
        
        User::factory()->create([
            'email' => 'subscribed@test.com',
            'subscription_status' => 'active',
            'stripe_customer_id' => 'cus_test123',
        ]);
    }
}
```

## CI/CD Testing Pipeline

### GitHub Actions Configuration

```yaml
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          
    steps:
      - uses: actions/checkout@v2
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          
      - name: Install Dependencies
        run: |
          composer install
          npm ci
          
      - name: Run Tests
        run: |
          php artisan test --parallel
          npm run test
          npx playwright test
          
      - name: Security Scan
        run: |
          composer audit
          npm audit
```

## Testing Commands

```bash
# Run all tests
php artisan test --parallel

# Run specific test suites
php artisan test --testsuite=Unit
php artisan test --testsuite=Feature

# Run with coverage
php artisan test --coverage --min=80

# Run specific test files
php artisan test tests/Unit/Scanners/SslTlsScannerTest.php

# Run E2E tests
npx playwright test
npx playwright test --headed  # With browser
npx playwright test --debug   # Debug mode

# Run security tests only
php artisan test --group=security

# Performance benchmarks
php artisan test --group=performance --stop-on-failure
```

## Test Coverage Requirements

- **Overall**: Minimum 80% coverage
- **Scanners**: 95% coverage (critical path)
- **Security**: 100% coverage (SSRF, validation)
- **API**: 90% coverage
- **Models**: 70% coverage
- **UI Components**: 60% coverage

## Common Testing Gotchas

1. **External Service Dependencies**: Always use real services for integration tests
2. **Async Job Testing**: Use `dispatchSync()` for synchronous testing
3. **WebSocket Testing**: Mock Reverb connections in unit tests
4. **Rate Limiting**: Clear cache between rate limit tests
5. **Time-based Tests**: Use `Carbon::setTestNow()` for consistent results