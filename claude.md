# Achilleus Claude Documentation

## Project Context
Achilleus is a production-ready security monitoring SaaS built with Laravel 12 and React 19. It provides comprehensive website security scanning with professional PDF reporting at $27/month for up to 10 domains, targeting freelance developers and small agencies.

## Quick Command Reference
```bash
# Use these shortcuts for task-based development:
achilleus plan 1.1    # Plan subtask 1.1 (see execution.md)
achilleus test 2.3    # Write tests for subtask 2.3 (see execution.md)
achilleus code 7.2    # Build subtask 7.2 (see execution.md)
achilleus verify 3.1  # Verify subtask 3.1 tests pass (see execution.md)

# Common Laravel commands:
php artisan serve     # Local development server
npm run dev          # Vite development server  
php artisan pest     # Run test suite
php artisan pint     # Format PHP code
```

## Development Best Practices

### Laravel 12 Best Practices
- **Use Service Classes**: Keep controllers thin, business logic in services
- **Type Hints Everywhere**: Leverage PHP 8.3+ features for better IDE support
- **Eloquent Relationships**: Use proper relationships, avoid N+1 queries with eager loading
- **Form Requests**: Validate input using dedicated Form Request classes
- **Resource Classes**: Transform model data consistently with API Resources
- **Jobs & Queues**: Use auto-scaling queues for all time-consuming operations

### Code Organization
```php
// Good: Service class with dependency injection
class SecurityScanService {
    public function __construct(
        private SecurityHeadersScanner $headersScanner,
        private SslTlsScanner $sslScanner,
        private DnsEmailScanner $dnsScanner,
        private SecurityScorer $scorer
    ) {}
}

// Good: Type-hinted controller with service injection
class DomainScanController extends Controller {
    public function store(ScanRequest $request, Domain $domain, SecurityScanService $scanner) {
        $this->authorize('scan', $domain);
        return $scanner->initiateScan($domain, $request->validated());
    }
}
```

### Database Best Practices
- **UUID Primary Keys**: Use UUIDs for better distributed scaling
- **JSONB for Scan Results**: Store scanner findings as structured JSONB
- **Proper Indexing**: Index frequently queried columns (user_id, created_at)
- **Relationships**: Define relationships properly for eager loading
- **Migrations**: Keep migrations small and reversible

### Security Best Practices
- **Input Validation**: Validate all user input with Form Requests
- **Authorization**: Use policies for all domain/scan access checks
- **SSRF Protection**: Validate URLs before making HTTP requests
- **Rate Limiting**: Implement scan rate limits (10/minute per user)
- **Secure Defaults**: Use secure HTTP client configurations

### Testing Best Practices
```php
// Good: Test with proper setup and assertions
test('user can scan owned domain', function () {
    $user = User::factory()->create();
    $domain = Domain::factory()->for($user)->create();
    
    $this->actingAs($user)
        ->post("/domains/{$domain->id}/scan")
        ->assertRedirect()
        ->assertSessionHas('success');
        
    expect(Scan::where('domain_id', $domain->id))->toHaveCount(1);
});

// Good: Mock external services in tests
test('ssl scanner handles connection failure gracefully', function () {
    Http::fake(['*' => Http::response('', 500)]);
    
    $scanner = new SslTlsScanner();
    $result = $scanner->scan('https://example.com');
    
    expect($result->status)->toBe('fail');
});
```

### Frontend Best Practices (React 19 + Inertia)
- **Component Composition**: Break UI into reusable components
- **TypeScript**: Use strict TypeScript for all components
- **shadcn/ui**: Use shadcn/ui components for consistency
- **Inertia Patterns**: Leverage Inertia's partial reloads and form handling
- **Error Boundaries**: Implement proper error handling

### Performance Best Practices
- **Query Optimization**: Use eager loading, avoid N+1 queries
- **Caching Strategy**: Cache dashboard metrics, scan results appropriately
- **Auto-scaling Queues**: Let Laravel Cloud handle queue scaling
- **Database Hibernation**: Leverage serverless PostgreSQL cost optimization
- **Edge Caching**: Use CloudFront for static assets

### API Design Best Practices
```php
// Good: Consistent API resource structure
class ScanResource extends JsonResource {
    public function toArray($request): array {
        return [
            'id' => $this->id,
            'status' => $this->status,
            'total_score' => $this->total_score,
            'grade' => $this->grade,
            'started_at' => $this->started_at,
            'completed_at' => $this->completed_at,
            'modules' => ScanModuleResource::collection($this->whenLoaded('modules')),
        ];
    }
}
```

## Technology Architecture

### Backend Stack
- **Laravel 12**: Core framework with React starter kit
- **PostgreSQL 15**: Auto-scaling serverless database with JSONB for scan results
- **Laravel Breeze**: Authentication backend only (no UI)
- **Redis**: Auto-scaling cache and session storage
- **Direct Stripe API**: Payment processing (no Cashier)

### Frontend Stack
- **React 19**: From Laravel 12 starter kit
- **Inertia.js 2**: SPA with server-side routing
- **TypeScript**: Type-safe development
- **Tailwind CSS 4**: Utility-first styling
- **shadcn/ui**: Professional React components
- **Vite**: Build tool and dev server

### Infrastructure (Laravel Cloud)
- **Hosting**: Laravel Cloud serverless platform
- **AWS Region**: US East (Ohio) us-east-2
- **Database**: Auto-scaling Serverless PostgreSQL
- **Cache**: Auto-scaling Redis
- **Storage**: S3 for PDF reports
- **CDN**: CloudFront edge caching
- **Queue Processing**: Auto-scaling Queue clusters (developer preview)
- **Auto-scaling**: 1-10 replicas with hibernation for cost optimization

## Core Features (Implemented)

### Security Scanning System
- **Three Scanners**: SSL/TLS (40%), Security Headers (30%), DNS/Email (30%)
- **Single Scan Type**: One comprehensive security analysis
- **Auto-scaling Queue Processing**: Laravel Cloud Queue clusters with automatic worker scaling
- **SSRF Protection**: NetworkGuard prevents internal network scanning
- **Smart Caching**: Redis with appropriate TTLs

### User Experience
- **Dark Theme UI**: #0a0a0b background throughout
- **4-Card Dashboard**: Score, Domains, Last Scan, Critical Issues
- **Collapsible Sidebar**: Dashboard → Domains → Activity → Reports
- **Trial System**: 14-day free trial with banner
- **Mobile-First**: Responsive design for all devices

### Business Features
- **Stripe Billing**: $27/month subscription
- **Domain Management**: CRUD with 10 domain limit
- **PDF Reports**: Professional branded reports
- **Activity History**: Complete scan tracking
- **Email Verification**: Required for activation

## Implementation Details

### Authentication Flow
```typescript
// Custom React components replace Breeze UI
// Backend uses Breeze authentication logic
pages/Auth/
├── Login.tsx        // Custom design, not Breeze
├── Register.tsx     // Custom design, not Breeze
└── ForgotPassword.tsx // Custom design, not Breeze
```

### Scanner Implementation
```php
// Weight distribution for scoring
SSL/TLS Scanner: 40%      // Certificate, ciphers, protocols
Security Headers: 30%     // HSTS, CSP, X-Frame-Options
DNS/Email Security: 30%   // SPF, DKIM, DMARC, DNSSEC
```

### Database Schema
```sql
users (UUID PK) → domains (UUID PK) → scans (UUID PK)
                                    → scan_modules (UUID PK)
                                    → reports (UUID PK)
                                    
JSONB storage for scan results
Optimized indexes for common queries
Auto-scaling with hibernation
```

## Laravel Cloud Queue Architecture

### Revolutionary Auto-scaling
**Previous Approach**: Manual Horizon configuration with fixed worker counts
```php
// OLD: Manual worker configuration in config/horizon.php
'production' => [
    'default' => [
        'minProcesses' => 2,
        'maxProcesses' => 20,
        // Manual guesswork for scaling
    ],
]
```

**New Laravel Cloud Approach**: Automatic scaling based on real-time metrics
- **Latency-Driven Scaling**: Adds workers when jobs queue too long
- **CPU/RAM Monitoring**: Tracks worker performance automatically
- **Job Throughput**: Monitors processing rates and backlog
- **Cost Optimization**: Scales down when demand decreases

### Queue Processing Benefits
- **Zero Configuration**: No manual worker tuning required
- **Real-time Monitoring**: Built-in metrics and dashboards
- **Perfect for Security Scans**: Handles varying scan loads efficiently
- **Cost Efficient**: Pay only for workers actually needed

## Development Tasks (Updated for Laravel Cloud)

### Task System Structure
Each task uses numbered subtasks (X.1, X.2, etc.) for granular tracking:
- **X.1**: Always planning/design
- **X.2-X.5**: Implementation steps
- **X.6**: Always testing/verification

### 🔷 Task 1: Email Notifications
```
1.1 Plan notification types (scan complete, issues found, trial ending)
1.2 Create notifications table and models
1.3 Build notification preferences UI in Profile
1.4 Implement Blade email templates with Achilleus branding
1.5 Add notification queue jobs (auto-scaling handles delivery)
1.6 Test email delivery and auto-scaling performance
```

### 🔷 Task 2: Enhanced Monitoring
```
2.1 Plan Laravel Cloud monitoring integration
2.2 Implement queue cluster metrics dashboard
2.3 Add hibernation cost tracking
2.4 Build auto-scaling performance analytics
2.5 Create alert system for queue failures
2.6 Test monitoring accuracy and scaling triggers
```

### 🔷 Task 3: API Access
```
3.1 Plan API endpoints and authentication (Sanctum)
3.2 Create personal access tokens with auto-scaling rate limits
3.3 Build API token management UI
3.4 Implement RESTful endpoints for domains/scans
3.5 Add queue-aware rate limiting and usage tracking
3.6 Test API performance under auto-scaling conditions
```

### 🔷 Task 4: Scheduled Scans
```
4.1 Design scheduling options (daily, weekly, monthly)
4.2 Create scan_schedules table and model
4.3 Build scheduling UI in domain settings
4.4 Implement Laravel scheduler with auto-scaling queues
4.5 Add notification for scheduled scan results
4.6 Test scheduling with queue cluster auto-scaling
```

### 🔷 Task 5: Export Features
```
5.1 Plan export formats (CSV, Excel, JSON)
5.2 Create export service with format handlers
5.3 Build export UI with format selection
5.4 Implement bulk export leveraging auto-scaling
5.5 Add export history tracking
5.6 Test export performance under varying loads
```

### 🔷 Task 6: Uptime Monitoring
```
6.1 Design uptime check architecture (separate from security)
6.2 Create uptime_checks table and models
6.3 Build uptime configuration UI
6.4 Implement monitoring workers with auto-scaling
6.5 Add downtime alerts and reporting
6.6 Test monitoring accuracy with queue clusters
```

### 🔷 Task 7: White-Label Support
```
7.1 Plan white-label features (logo, colors, domain)
7.2 Create white_label_configs table
7.3 Build white-label settings UI
7.4 Implement dynamic theming system
7.5 Add custom domain support with Laravel Cloud SSL
7.6 Test multiple white-label instances
```

### 🔷 Task 8: Advanced Scanner Options
```
8.1 Plan additional scanners (performance, accessibility)
8.2 Create modular scanner plugin system
8.3 Build scanner selection UI
8.4 Implement new scanner modules with auto-scaling
8.5 Add scanner-specific scoring weights
8.6 Test new scanners and auto-scaling performance
```

## Key Technical Decisions

### Why Laravel Cloud Over Traditional Hosting
- **Zero DevOps**: No server management or scaling decisions
- **Cost Efficiency**: Pay-per-use with hibernation
- **Auto-scaling Queues**: Eliminates manual worker configuration
- **Built for Laravel**: Optimized specifically for Laravel applications
- **Enterprise Security**: WAF, DDoS protection, automatic SSL included

### What We Avoided
- **Manual Queue Configuration**: Laravel Cloud handles auto-scaling
- **Traditional VPS**: Cloud manages everything automatically
- **Custom Components**: shadcn/ui provides consistency
- **Uptime Monitoring**: Not part of security focus (yet)
- **Multiple Scan Types**: Single comprehensive scan simpler

## Production Status

### ✅ Fully Implemented
- Complete authentication system with email verification
- Domain CRUD with validation and limits
- Three-scanner security analysis system
- Auto-scaling queue processing with Laravel Cloud
- Professional PDF report generation
- Stripe subscription billing
- 4-card dashboard with trends
- Activity history and filtering
- Mobile-responsive dark theme UI
- Laravel Cloud deployment with hibernation

### 🔄 Currently Disabled (via Pennant)
- Client management features
- Automation workflows
- Team functionality

### 📊 Success Metrics
- **Target**: 300 customers @ $27 = $8,100 MRR
- **Infrastructure Cost**: Optimized through hibernation and auto-scaling
- **Trial Conversion**: Aiming for 30%
- **Churn Rate**: Target <5% monthly
- **Queue Performance**: >95% success rate with automatic scaling

## Critical Implementation Notes

### Security Considerations
- **SSRF Protection**: NetworkGuard blocks internal IPs
- **Rate Limiting**: 10 scans/minute per user
- **Input Validation**: Form requests for all user input
- **XSS Protection**: React auto-escaping
- **SQL Injection**: Eloquent ORM parameterization

### Performance Optimizations
- **Auto-scaling Queues**: No manual worker tuning required
- **Serverless Database**: Automatic performance scaling
- **Query Caching**: 15-minute TTL for dashboard
- **Eager Loading**: Prevent N+1 queries
- **CDN Assets**: CloudFront edge caching

### Laravel Cloud Benefits
- **Zero DevOps**: No server management
- **Auto-scaling**: Handles traffic spikes automatically
- **Hibernation**: Saves costs when idle
- **Managed Services**: Database, cache, storage, queues
- **Automatic SSL**: Certificates handled
- **Global CDN**: Fast asset delivery

## Quick Reference

### File Structure
```
resources/js/Pages/     # React page components
app/Http/Controllers/   # Inertia controllers
app/Services/          # Business logic
app/Jobs/             # Auto-scaling queue jobs
app/Models/           # Eloquent models
app/Policies/         # Authorization
```

### Common Commands
```bash
# Laravel Cloud auto-manages queue processing
npm run dev             # Start Vite dev server
php artisan pest        # Run test suite
php artisan pint        # Format PHP code
npm run build          # Production build
```

### Environment Keys
```env
# Laravel Cloud auto-injects DB/Redis/S3
STRIPE_KEY=pk_live_xxx
STRIPE_SECRET=sk_live_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
STRIPE_PRICE_ID=price_xxx
```

## Current Development Focus
The application is **production-ready** with auto-scaling infrastructure. Next priorities should focus on growth features like email notifications (Task 1) and enhanced monitoring (Task 2) to leverage Laravel Cloud's new queue cluster capabilities. The auto-scaling queues eliminate the need for manual worker configuration, allowing focus on business features rather than infrastructure management.