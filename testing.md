# Achilleus Testing Strategy

**Version**: 1.0  
**Test Target**: 365 tests maximum  
**Coverage Goal**: 80% on business logic  
**Execution Time**: <30 seconds with parallel testing

---

## Testing Philosophy

### Core Principles
1. **Test Business Rules, Not Framework**: Laravel is already tested
2. **Mock External Services**: Never hit real APIs in tests
3. **Quality Over Quantity**: 365 solid tests > 500 fragile tests
4. **Fast Feedback Loop**: Tests must run quickly
5. **Pragmatic Coverage**: 80% is enough, 100% has diminishing returns

### What We Test
- ✅ Business logic (domain limits, trial expiry, scoring)
- ✅ API contracts (request validation, response format)
- ✅ Security rules (SSRF protection, authorization)
- ✅ Payment flows (subscription lifecycle)
- ✅ Critical user paths (signup → scan → payment)

### What We DON'T Test
- ❌ Laravel framework features
- ❌ External API responses (mock instead)
- ❌ Database queries (trust Eloquent)
- ❌ UI styling and animations
- ❌ Third-party package internals
- ❌ Getter/setter methods
- ❌ Configuration files

---

## Test Distribution (365 Total)

### Phase-by-Phase Allocation

| Phase | Focus | Tests | Description |
|-------|-------|-------|-------------|
| Phase 1 | Project Setup & Auth | 20 | Registration, login, middleware |
| Phase 2 | Database & Models | 30 | Business logic, relationships |
| Phase 3 | Security Infrastructure | 25 | SSRF, NetworkGuard, AbstractScanner |
| Phase 4 | SSL/TLS Scanner | 30 | Certificate analysis, validation |
| Phase 5 | Security Headers Scanner | 25 | HTTP headers, CSP, HSTS |
| Phase 6 | DNS/Email Scanner | 30 | SPF, DKIM, DMARC, DNSSEC |
| Phase 7 | Scan Orchestration | 30 | Job queue, weighted scoring |
| Phase 8 | Core UI | 30 | Dashboard, domain list |
| Phase 9 | Domain Detail | 25 | Scan results, configuration |
| Phase 10 | Settings & Profile | 25 | User management, activity |
| Phase 10.5 | Email Notifications | 15 | Certificate/trial expiry warnings |
| Phase 11 | Report Generation | 25 | PDF creation, S3 storage |
| Phase 12 | OAuth & Social Login | 15 | Google OAuth integration |
| Phase 13 | Payment & Billing | 30 | Stripe, subscriptions |
| Phase 14 | Landing Page | 15 | Marketing site, SEO |
| Phase 15 | Production Polish | 20 | Deployment, monitoring |

### Test Type Breakdown
- **Unit Tests**: 120 (isolated business logic)
- **Feature Tests**: 195 (API and integration)
- **Browser Tests**: 50 (critical E2E paths)

---

## Mocking Strategy

### Always Mock These Services

```php
// Stripe Payments
use Laravel\Cashier\SubscriptionBuilder;

beforeEach(function () {
    $this->mock(SubscriptionBuilder::class)
        ->shouldReceive('create')
        ->andReturn(Subscription::factory()->make());
});
```

```php
// OAuth Providers
use Laravel\Socialite\Facades\Socialite;

Socialite::shouldReceive('driver->user')
    ->andReturn((object) [
        'id' => '12345',
        'email' => 'test@example.com',
        'name' => 'Test User'
    ]);
```

```php
// External HTTP Requests
Http::fake([
    // Mock SSL scanner responses
    'https://*' => Http::response([], 200, [
        'Strict-Transport-Security' => 'max-age=31536000',
        'X-Frame-Options' => 'DENY'
    ]),
    
    // Mock DNS lookups
    'dns-api.org/*' => Http::response([
        'Answer' => [
            ['data' => 'v=spf1 include:_spf.google.com ~all']
        ]
    ])
]);
```

```php
// Email Sending
Mail::fake();

// After action that sends email
Mail::assertSent(WelcomeEmail::class, function ($mail) use ($user) {
    return $mail->hasTo($user->email);
});
```

```php
// Queue Jobs
Queue::fake();

// After action that dispatches job
Queue::assertPushed(RunDomainScan::class, function ($job) use ($domain) {
    return $job->domain->id === $domain->id;
});
```

---

## Test Patterns

### 1. Model Business Logic Tests

```php
// tests/Unit/Models/UserTest.php
use App\Models\User;

test('user trial expires after 14 days', function () {
    $user = User::factory()->create([
        'created_at' => now()->subDays(15),
        'trial_ends_at' => now()->subDay(),
    ]);
    
    expect($user->isInTrial())->toBeFalse();
    expect($user->trialDaysRemaining())->toBe(0);
});

test('user cannot add more than 10 domains', function () {
    $user = User::factory()
        ->has(Domain::factory()->count(10))
        ->create();
    
    expect($user->canAddDomain())->toBeFalse();
    
    expect(fn() => $user->domains()->create([
        'url' => 'example.com'
    ]))->toThrow(DomainLimitException::class);
});
```

### 2. API Endpoint Tests

```php
// tests/Feature/Api/DomainApiTest.php
test('creating domain validates HTTPS requirement', function () {
    $user = User::factory()->create();
    
    $response = $this->actingAs($user)
        ->postJson('/api/domains', [
            'url' => 'http://example.com', // HTTP not allowed
        ]);
    
    $response->assertStatus(422)
        ->assertJsonValidationErrors(['url']);
});

test('domain ownership is enforced on deletion', function () {
    $owner = User::factory()->create();
    $otherUser = User::factory()->create();
    $domain = Domain::factory()->for($owner)->create();
    
    $response = $this->actingAs($otherUser)
        ->deleteJson("/api/domains/{$domain->id}");
    
    $response->assertForbidden();
});
```

### 3. Scanner Tests (Mocked)

```php
// tests/Unit/Scanners/SecurityScannerTest.php
test('scanner calculates weighted score correctly', function () {
    Http::fake(); // Never hit real sites
    
    $orchestrator = app(ScanOrchestrator::class);
    
    // Mock scanner results
    $results = collect([
        new ModuleResult('ssl_tls', 80, 'ok'),      // 80 * 0.4 = 32
        new ModuleResult('security_headers', 90, 'ok'), // 90 * 0.3 = 27
        new ModuleResult('dns_email', 70, 'warn'),  // 70 * 0.3 = 21
    ]);
    
    $score = $orchestrator->calculateScore($results);
    
    expect($score)->toBe(80); // 32 + 27 + 21 = 80
});

test('SSRF protection blocks private IPs', function () {
    $guard = app(NetworkGuard::class);
    
    $privateIPs = [
        'https://192.168.1.1',
        'https://10.0.0.1',
        'https://localhost',
        'https://127.0.0.1',
        'https://169.254.169.254', // AWS metadata
    ];
    
    foreach ($privateIPs as $ip) {
        expect(fn() => $guard->assertPublicHttpUrl($ip))
            ->toThrow(SecurityException::class);
    }
});

test('DNSSEC detection correctly validates enabled domains', function () {
    $scanner = app(DnsEmailScanner::class);
    
    // Mock successful DNSSEC validation with both DNSKEY and DS records
    $mockResult = [
        'enabled' => true,
        'chain_valid' => true,
        'algorithms' => ['8', '13'], // RSA/SHA-256 and ECDSA
        'method' => 'comprehensive'
    ];
    
    $scanner->shouldReceive('validateDNSSECChain')
        ->with('cloudflare.com')
        ->andReturn($mockResult);
    
    $result = $scanner->checkDNSSEC('cloudflare.com');
    
    expect($result['score'])->toBe(100);
    expect($result['details']['dnssec']['enabled'])->toBeTrue();
    expect($result['details']['dnssec']['chain_valid'])->toBeTrue();
    expect($result['details']['strengths'])->toContain('DNSSEC enabled with valid trust chain');
});

test('DNSSEC detection handles false negatives correctly', function () {
    $scanner = app(DnsEmailScanner::class);
    
    // Mock scenario where DNSSEC is enabled but trust chain validation fails
    $mockResult = [
        'enabled' => true,
        'chain_valid' => false,
        'algorithms' => ['8'],
        'method' => 'comprehensive'
    ];
    
    $scanner->shouldReceive('validateDNSSECChain')
        ->with('example.com')
        ->andReturn($mockResult);
    
    $result = $scanner->checkDNSSEC('example.com');
    
    expect($result['score'])->toBe(85); // 100 - 15 penalty for invalid chain
    expect($result['details']['dnssec']['enabled'])->toBeTrue();
    expect($result['details']['issues'])->toContain('DNSSEC enabled but trust chain validation failed');
});

test('DNSSEC detection uses fallback methods when primary fails', function () {
    $scanner = app(DnsEmailScanner::class);
    
    // Mock fallback scenario
    $mockResult = [
        'enabled' => true,
        'chain_valid' => false,
        'algorithms' => [],
        'method' => 'fallback'
    ];
    
    $scanner->shouldReceive('validateDNSSECChain')
        ->with('fallback-test.com')
        ->andReturn($mockResult);
    
    $result = $scanner->checkDNSSEC('fallback-test.com');
    
    expect($result['details']['dnssec']['validation_method'])->toBe('fallback');
    expect($result['score'])->toBeGreaterThan(0);
});

test('DNSSEC validation methods are properly layered', function () {
    $scanner = app(DnsEmailScanner::class);
    
    // Test that validation tries multiple methods in correct order
    $scanner->shouldReceive('queryWithDNSSECValidation')
        ->with('test.com', DNS_DNSKEY)
        ->andReturn(['found' => true, 'algorithms' => ['8']]);
    
    $scanner->shouldReceive('queryDSRecords')
        ->with('test.com')
        ->andReturn(['found' => true, 'algorithms' => ['8']]);
    
    $scanner->shouldReceive('validateTrustChain')
        ->with('test.com')
        ->andReturn(['valid' => true]);
    
    $result = $scanner->validateDNSSECChain('test.com');
    
    expect($result['enabled'])->toBeTrue();
    expect($result['chain_valid'])->toBeTrue();
    expect($result['method'])->toBe('comprehensive');
});

test('DNSSEC timeout handling prevents false negatives', function () {
    $scanner = app(DnsEmailScanner::class);
    
    // Verify extended timeout is used for DNSSEC queries
    expect($scanner->getTimeout())->toBeGreaterThan(30);
    
    // Mock timeout scenario - should not immediately fail to 'not enabled'
    $scanner->shouldReceive('validateDNSSECChain')
        ->with('slow-dns.com')
        ->andThrow(new \Exception('DNS query timeout'));
    
    $scanner->shouldReceive('basicDNSSECCheck')
        ->with('slow-dns.com')
        ->andReturn(['enabled' => true]);
    
    $result = $scanner->checkDNSSEC('slow-dns.com');
    
    // Should still detect DNSSEC via fallback, not immediately assume disabled
    expect($result['details']['dnssec']['validation_method'])->toBe('basic');
});

test('DNS scanner prevents DNSSEC false negatives from resolver issues', function () {
    $scanner = app(DnsEmailScanner::class);
    
    // Test multiple query methods to prevent resolver-specific issues
    $scanner->shouldReceive('queryWithDNSSECValidation')
        ->andReturn(['found' => false, 'algorithms' => []]); // dig fails
    
    $scanner->shouldReceive('queryDSRecords')
        ->andReturn(['found' => false, 'algorithms' => []]); // DS lookup fails
    
    // But fallback methods should catch it
    $scanner->shouldReceive('fallbackDNSSECCheck')
        ->andReturn(['enabled' => true]);
    
    $result = $scanner->validateDNSSECChain('dnssec-domain.com');
    
    expect($result['enabled'])->toBeTrue();
    expect($result['method'])->toBe('fallback');
});
```

### 4. Payment Tests

```php
// tests/Feature/Billing/SubscriptionTest.php
test('trial converts to paid subscription', function () {
    $user = User::factory()->withTrial()->create();
    
    // Mock Stripe
    $this->mock(SubscriptionBuilder::class)
        ->shouldReceive('create')
        ->once()
        ->andReturn(Subscription::factory()->make());
    
    $response = $this->actingAs($user)
        ->postJson('/api/subscription', [
            'payment_method' => 'pm_card_visa',
        ]);
    
    $response->assertSuccessful();
    expect($user->fresh()->subscribed())->toBeTrue();
});

test('expired trial blocks feature access', function () {
    $user = User::factory()->create([
        'trial_ends_at' => now()->subDay(),
    ]);
    
    $domain = Domain::factory()->for($user)->create();
    
    $response = $this->actingAs($user)
        ->postJson("/api/domains/{$domain->id}/scan");
    
    $response->assertStatus(402) // Payment Required
        ->assertJson(['message' => 'Subscription required']);
});
```

### 5. E2E Critical Path Test

```php
// tests/Browser/UserJourneyTest.php
use Laravel\Dusk\Browser;

test('complete user journey from signup to scan', function () {
    $this->browse(function (Browser $browser) {
        $browser->visit('/register')
            ->type('email', 'test@example.com')
            ->type('password', 'password')
            ->press('Sign Up')
            ->assertPathIs('/dashboard')
            ->assertSee('14 days remaining')
            
            ->click('@add-domain')
            ->type('url', 'https://example.com')
            ->press('Add Domain')
            ->waitForText('Domain added')
            
            ->click('@scan-now')
            ->waitForText('Scan completed', 35)
            ->assertSee('Security Score');
    });
});
```

---

## Testing Commands

### Daily Development
```bash
# Run tests for current feature
php artisan test --filter=DomainTest

# Run with coverage check
php artisan test --coverage --min=80

# Run fast with parallel execution
php artisan test --parallel

# Stop on first failure (debugging)
php artisan test --stop-on-failure
```

### Pre-Commit Checklist
```bash
# 1. Run all tests
php artisan test

# 2. Check coverage
php artisan test --coverage

# 3. Run linting
./vendor/bin/pint

# 4. Clear test cache if needed
php artisan test --recreate-databases
```

---

## Factory Patterns

### User Factory States
```php
// database/factories/UserFactory.php
public function withTrial(): static
{
    return $this->state(fn (array $attributes) => [
        'trial_ends_at' => now()->addDays(14),
    ]);
}

public function withExpiredTrial(): static
{
    return $this->state(fn (array $attributes) => [
        'trial_ends_at' => now()->subDay(),
    ]);
}

public function withSubscription(): static
{
    return $this->state(fn (array $attributes) => [
        'stripe_customer_id' => 'cus_' . Str::random(14),
        'subscription_status' => 'active',
    ]);
}
```

### Domain Factory States
```php
// database/factories/DomainFactory.php
public function scanned(): static
{
    return $this->state(fn (array $attributes) => [
        'last_scan_at' => now(),
        'last_scan_score' => fake()->numberBetween(60, 100),
    ]);
}
```

---

## Performance Optimization

### Speed Up Test Suite
1. **Use SQLite in-memory for tests**
   ```env
   # .env.testing
   DB_CONNECTION=sqlite
   DB_DATABASE=:memory:
   ```

2. **Run tests in parallel**
   ```bash
   php artisan test --parallel --processes=4
   ```

3. **Use `RefreshDatabase` trait efficiently**
   ```php
   uses(RefreshDatabase::class)->beforeEach(function () {
       // Runs migrations once per test class
   });
   ```

4. **Share expensive setups**
   ```php
   beforeAll(function () {
       // Expensive setup that runs once
   });
   ```

---

## Common Pitfalls to Avoid

### ❌ Don't Test External APIs
```php
// BAD - Slow, flaky, rate-limited
$result = $scanner->scan('https://example.com');

// GOOD - Fast, reliable, controlled
Http::fake(['example.com' => Http::response([...])]);
$result = $scanner->scan('https://example.com');
```

### ❌ Don't Over-Test Simple Methods
```php
// BAD - Testing framework features
test('user has domains relationship', function () {
    $user = User::factory()->create();
    expect($user->domains())->toBeInstanceOf(HasMany::class);
});

// GOOD - Test business logic instead
test('user domains scope returns only active', function () {
    $user = User::factory()->create();
    Domain::factory()->for($user)->active()->count(3)->create();
    Domain::factory()->for($user)->inactive()->count(2)->create();
    
    expect($user->domains()->active()->count())->toBe(3);
});
```

### ❌ Don't Create Test Dependencies
```php
// BAD - Tests depend on execution order
test('create user', function () {
    $this->user = User::factory()->create();
});

test('user can add domain', function () {
    $this->user->domains()->create([...]); // Fails if first test didn't run
});

// GOOD - Each test is independent
test('user can add domain', function () {
    $user = User::factory()->create();
    $domain = $user->domains()->create([...]);
    expect($domain)->toExist();
});
```

---

## CI/CD Integration
```yaml
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v2
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          
      - name: Install Dependencies
        run: |
          composer install
          npm ci && npm run build
          
      - name: Run Tests
        env:
          DB_CONNECTION: sqlite
          DB_DATABASE: :memory:
        run: |
          php artisan test --parallel
          
      - name: Check Coverage
        run: |
          php artisan test --coverage --min=80
```

---

## Summary

This testing strategy ensures:
- **Complete coverage** of business-critical features
- **Fast execution** through mocking and parallel testing
- **Maintainable tests** that don't break with minor changes
- **Clear boundaries** on what to test and what to skip

Remember: **365 quality tests** that run in 30 seconds are infinitely better than 500 flaky tests that take 10 minutes. Focus on business value, not coverage percentage.