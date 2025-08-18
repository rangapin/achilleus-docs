# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Achilleus** - Security monitoring SaaS for developers managing multiple websites  
**Target**: Freelancers and small businesses needing affordable security monitoring  
**Price**: $27/month for 10 domains with unlimited scans  
**Stack**: Laravel 12 (with React starter kit) + React 19 + Shadcn/ui + Inertia.js + Laravel Reverb + Laravel Cloud
**Support**: security@achilleus.so (no automated email system in MVP)

## Key Resources
- **Laravel 12**: https://laravel.com/docs/12.x
- **Laravel Starter Kits**: https://laravel.com/docs/12.x/starter-kits (we use React kit)
- **React 19**: https://react.dev/
- **Laravel Cloud**: https://cloud.laravel.com/
- **Shadcn/ui**: https://ui.shadcn.com/
- **Laravel Reverb**: https://reverb.laravel.com/
- **Landing Page**: Salient template from https://tailwindcss.com/plus/templates/salient  

## Documentation Structure

- `technical.md` - System architecture, scanner implementation, database schema
- `product.md` - Product specifications, UI mockups, business rules  
- `execution.md` - Development phases, testing strategy, deployment plan

## Development Workflow Shortcuts

### Core Commands

```bash
# Planning Phase
achilleus-plan() {
    echo "📋 PLANNING: Break down → Security review → Test cases → SSRF validation"
}

# Testing Phase (TDD)
achilleus-test() {
    echo "🧪 TESTS FIRST: Unit → Feature → E2E → Security validation"
    echo "Commands: php artisan pest | npx playwright test"
}

# Implementation Phase
achilleus-code() {
    echo "💻 CODE: Laravel conventions → Shadcn/ui → Security-first → Error handling"
}

# Verification Phase
achilleus-verify() {
    echo "✅ VERIFY: php artisan pest --parallel && npm run type-check && npx playwright test"
}
```

## Security-First Development (Critical)

### S1: Input Validation & SSRF Protection
```php
// ALWAYS use Form Requests
class StoreDomainRequest extends FormRequest {
    public function rules(): array {
        return [
            'url' => ['required', 'url', 'starts_with:https://', new PublicUrl()],
        ];
    }
}

// ALWAYS validate URLs before external requests
NetworkGuard::assertPublicHttpUrl($url);
```

### S2: Scanner Implementation Pattern
```php
class SecurityScanner extends AbstractScanner {
    protected int $timeout = 20;
    protected int $retryAttempts = 2;
    
    // Base class handles SSRF, retries, rate limiting automatically
    protected function performScan(string $url, array $context = []): ModuleResult {
        // Your scanner logic here
    }
}

// Jobs are dispatched to Laravel Cloud managed workers
RunDomainScan::dispatch($domain); // Cloud handles queue infrastructure
```

### S3: Rate Limiting & Authentication
```php
// Two-tier rate limiting: User level (10 scans/minute)
Route::middleware(['throttle:scans'])->group(function () {
    Route::post('/domains/{domain}/scan', [DomainScanController::class, 'store']);
});

// Scanner level rate limiting handled in AbstractScanner
// Always authorize domain access
$this->authorize('view', $domain);
```

## Best Practices Checklist

### BP1: Laravel Patterns
- [ ] Use Form Requests for all validation
- [ ] Implement proper authorization policies  
- [ ] Use Eloquent relationships correctly
- [ ] Apply middleware consistently
- [ ] Handle errors gracefully with try/catch

### BP2: React + Shadcn/ui
- [ ] Use TypeScript interfaces consistently
- [ ] Implement React Hook Form + Zod validation
- [ ] Apply Shadcn/ui components (Button, Card, Form, Table, Dialog)
- [ ] Handle loading states and errors
- [ ] Ensure dark theme consistency

### BP3: Database & Models
- [ ] Create migrations before models
- [ ] Define relationships in models
- [ ] Use appropriate data types (JSONB for scan results)
- [ ] Add proper indexes for performance
- [ ] Implement soft deletes where needed

### BP4: Testing Strategy
- [ ] Write tests BEFORE implementation (TDD)
- [ ] Unit tests for scanners and services
- [ ] Feature tests for API endpoints
- [ ] E2E tests with Playwright for critical flows
- [ ] Test security validation and SSRF protection

## Code Quality Standards

### C1: Security Requirements
- HTTPS only domains (no HTTP allowed)
- NetworkGuard validation on all external requests
- Input validation on all user inputs
- Rate limiting on scan operations
- SSRF protection against private IPs and localhost

### C2: Performance Standards
- Dashboard load time < 500ms
- Scan completion < 30 seconds
- API responses < 200ms
- Database queries optimized with proper indexes
- Caching implemented for frequently accessed data

### C3: UI/UX Standards
- Consistent dark theme (#0a0a0b background)
- Mobile-responsive design
- Loading states for all async operations
- Empty states with actionable guidance
- Error messages with clear next steps

### C4: Code Organization
- Follow Laravel directory structure
- Use service classes for business logic
- Keep controllers thin (max 20 lines per method)
- Extract reusable components
- Document complex business logic

## Scanner Architecture (Core Feature)

### Scanner Types & Weights
- **SSL/TLS Scanner**: 40% weight - Certificate validation, protocol analysis
- **Security Headers Scanner**: 30% weight - HSTS, CSP, security headers
- **DNS/Email Scanner**: 30% weight - SPF, DKIM, DMARC validation

### Implementation Requirements
```php
abstract class AbstractScanner implements Scanner {
    // Common functionality: rate limiting, SSRF protection, retry logic
    public function scan(string $url, array $context = []): ModuleResult;
    abstract protected function performScan(string $url, array $context = []): ModuleResult;
}
```

### Security Scoring
- Grades: A+ (95-100), A (90-94), B+ (85-89), B (80-84), C (70-79), D (60-69), F (0-59)
- Weighted calculation: Total = (SSL×0.4) + (Headers×0.3) + (DNS×0.3)
- When a scanner fails: Redistribute weights proportionally among working scanners
- Store raw scan data in JSONB for detailed analysis

## Database Schema (Essential)

### Core Tables
```sql
users: id, email, trial_ends_at, subscription_status
domains: id, user_id, url, email_mode, last_scan_score, last_scanned_at
scans: id, domain_id, user_id, status, total_score, grade, started_at, completed_at
scan_modules: id, scan_id, module, score, status, raw JSONB NOT NULL DEFAULT '{}'
reports: id, user_id, filename, file_path, generated_at
```

### Key Relationships
- User hasMany Domains, Scans, Reports
- Domain belongsTo User, hasMany Scans
- Scan belongsTo Domain/User, hasMany ScanModules

## API Design Patterns

### Standard CRUD Controllers
```php
class DomainController extends Controller {
    public function index(): Response     // GET /domains
    public function store(StoreDomainRequest $request): Response  // POST /domains
    public function show(Domain $domain): Response    // GET /domains/{id}
    public function destroy(Domain $domain): Response // DELETE /domains/{id}
}
```

### Scan Operations
```php
class DomainScanController extends Controller {
    public function store(Domain $domain): Response {
        // Create scan record, dispatch job, return scan ID
        $scan = $domain->scans()->create(['status' => 'pending']);
        RunDomainScan::dispatch($scan);
        return response()->json(['scan_id' => $scan->id]);
    }
}
```

## Frontend Patterns

### Page Structure (Inertia.js)
```typescript
// Standard page component structure
export default function DomainsIndex({ domains, auth }: PageProps<{ domains: Domain[] }>) {
    return (
        <AuthenticatedLayout user={auth.user}>
            <Head title="Domains" />
            <PageHeader title="Domains" actions={<AddDomainButton />} />
            <DomainsTable domains={domains} />
        </AuthenticatedLayout>
    );
}
```

### Form Handling
```typescript
// Use React Hook Form + Zod for all forms
const form = useForm<z.infer<typeof domainSchema>>({
    resolver: zodResolver(domainSchema),
    defaultValues: { email_mode: "expected" }
});

const onSubmit = (values: z.infer<typeof domainSchema>) => {
    router.post('/domains', values);
};
```

## Testing Approach

### Unit Tests (Pest PHP)
```php
test('ssl scanner detects expired certificates', function () {
    $scanner = app(SslTlsScanner::class);
    $result = $scanner->scan('https://expired.badssl.com');
    expect($result->score)->toBe(0);
});
```

### E2E Tests (Playwright)
```typescript
test('user can add domain and view scan results', async ({ page }) => {
    await page.goto('/domains');
    await page.getByRole('button', { name: 'Add Domain' }).click();
    await page.fill('[name="url"]', 'https://example.com');
    await page.getByRole('button', { name: 'Add & Scan' }).click();
    await expect(page.getByText('Scan completed')).toBeVisible();
});
```

## Deployment & Production

### Laravel Cloud Configuration
- **URL**: https://cloud.laravel.com/
- Database: PostgreSQL with hibernation (managed)
- Queue: Redis with auto-scaling workers (no manual config needed)
- Storage: S3 for PDF reports
- Monitoring: Built-in application performance monitoring
- Real-time: Laravel Reverb for WebSocket connections
- Workers: Automatic process management and restarts

### Environment Variables
```env
# Core Configuration
APP_ENV=production
APP_DEBUG=false
APP_URL=https://achilleus.app

# Database
DB_CONNECTION=pgsql
DB_DATABASE=achilleus_production

# Queue & Cache (Laravel Cloud managed)
QUEUE_CONNECTION=redis  # Cloud provides Redis
CACHE_DRIVER=redis      # Auto-configured by Cloud

# Stripe
STRIPE_KEY=pk_live_...
STRIPE_SECRET=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Security
BCRYPT_ROUNDS=12
SESSION_SECURE_COOKIE=true
```

## Common Commands

### Development
```bash
# Local Development
php artisan migrate --seed
php artisan queue:work  # Local only, Cloud handles this in production
npm run dev

# Testing
php artisan pest --parallel
npm run type-check
npx playwright test

# Quality
php artisan pint
npm run lint

# Production
php artisan optimize
php artisan config:cache
php artisan route:cache
```

### Security Testing
```bash
# Test SSRF protection
curl -X POST -d '{"url":"http://localhost"}' /api/domains
curl -X POST -d '{"url":"http://192.168.1.1"}' /api/domains
curl -X POST -d '{"url":"http://169.254.169.254"}' /api/domains

# Test rate limiting
for i in {1..15}; do curl -H "Authorization: Bearer $token" /api/domains/1/scan; done
```

## Critical Security Reminders

1. **NEVER** make HTTP requests without NetworkGuard validation
2. **ALWAYS** validate user input with Form Requests
3. **NEVER** trust external data without sanitization
4. **ALWAYS** use HTTPS-only domain validation
5. **NEVER** log sensitive data (URLs, API keys, personal info)
6. **ALWAYS** implement proper rate limiting
7. **NEVER** expose stack traces in production
8. **ALWAYS** authorize domain access with policies

## Support & Debugging

### Common Issues
- **SSRF Blocked**: Check NetworkGuard configuration, ensure public URLs only
- **Scan Timeouts**: Verify timeout settings in scanner classes
- **Queue Jobs Failing**: Check failed_jobs table, Laravel Cloud logs, verify scanner dependencies
- **Rate Limiting**: Monitor cache keys, adjust limits in RateLimiter config
- **Support Requests**: Direct users to security@achilleus.so (no automated email in MVP)

### Logging
```php
// Security events
Log::warning('SSRF attempt blocked', ['url' => $url]);

// Scanner issues  
Log::error('SSL scan failed', ['domain' => $domain->url, 'error' => $e->getMessage()]);

// Performance monitoring
Log::info('Scan completed', ['duration' => $duration, 'score' => $score]);
```

This streamlined CLAUDE.md provides essential guidance in under 500 lines while maintaining all critical information for building Achilleus securely and efficiently.