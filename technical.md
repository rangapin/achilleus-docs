# Achilleus Technical Documentation

## Technology Stack

### Backend Framework
- **Laravel 12**: Primary application framework with React starter kit
- **PostgreSQL 15**: Auto-scaling serverless database with JSONB support
- **Laravel Breeze**: Authentication scaffolding only (no UI components used)
- **Laravel Pennant**: Feature flag management for controlled rollouts
- **Redis**: Auto-scaling caching and session storage

### Frontend Architecture
- **Laravel 12 React Starter Kit**: Modern React/Inertia foundation
- **React 19**: Latest React from Laravel starter kit
- **Inertia.js 2**: SPA experience with server-side routing
- **TypeScript**: Type-safe frontend development
- **Tailwind CSS 4**: Utility-first styling framework
- **shadcn/ui**: React component library for consistent UI
- **Vite**: Lightning-fast build tool and dev server

### Laravel Starter Kit Implementation
```bash
# Project initialized with (Subtask 1.1):
laravel new achilleus --react --inertia
# Selected: React starter kit with Inertia 2
# Authentication: Laravel Breeze (backend only)
# Frontend: Custom UI replacing Breeze components
```

### Core Dependencies
```json
{
  "backend": {
    "laravel/pennant": "Feature flags",
    "laravel/pulse": "Application monitoring", 
    "spatie/laravel-activitylog": "Activity tracking",
    "barryvdh/laravel-dompdf": "PDF generation",
    "predis/predis": "Redis client",
    "stripe/stripe-php": "Payment processing"
  },
  "frontend": {
    "@inertiajs/react": "Inertia React adapter",
    "@shadcn/ui": "UI components",
    "react": "^19.0.0",
    "typescript": "^5.0.0", 
    "tailwindcss": "^4.0.0",
    "recharts": "Charts and graphs",
    "lucide-react": "Icon library"
  },
  "development": {
    "pestphp/pest": "Testing framework",
    "laravel/pint": "Code formatting",
    "laravel/telescope": "Debug assistance"
  }
}
```

## Database Architecture (Subtasks 1.2, 4.1)

### Core Tables Structure

#### Users Table (Subtask 1.2)
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    email_verified_at TIMESTAMP NULL,
    password VARCHAR(255) NOT NULL,
    trial_ends_at TIMESTAMP NOT NULL,
    stripe_customer_id VARCHAR(255) NULL,
    stripe_subscription_id VARCHAR(255) NULL,
    subscription_status VARCHAR(50) DEFAULT 'trialing',
    onboarding_completed BOOLEAN DEFAULT FALSE,
    timezone VARCHAR(50) DEFAULT 'UTC',
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    INDEX idx_trial_status (trial_ends_at, subscription_status)
);
```

#### Enhanced Domains Table (Subtasks 1.2, 4.1)
```sql
CREATE TABLE domains (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    url VARCHAR(255) NOT NULL,
    display_name VARCHAR(255) NOT NULL,
    platform VARCHAR(50) NULL,
    email_mode VARCHAR(10) DEFAULT 'expected', -- expected|none (Subtask 2.4)
    dkim_selector VARCHAR(50) NULL,           -- Custom DKIM selector (Subtask 2.4)
    last_scan_id UUID NULL,
    last_scan_at TIMESTAMP NULL,
    last_scan_score SMALLINT NULL,
    security_score INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    INDEX idx_user_domains (user_id, is_active),
    INDEX idx_last_scan (last_scan_at DESC)
);
```

#### Comprehensive Scans Table (Subtasks 1.2, 2.5, 3.2)
```sql
CREATE TABLE scans (
    id UUID PRIMARY KEY,
    domain_id UUID REFERENCES domains(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE, -- For direct user queries
    status ENUM('pending', 'running', 'completed', 'failed'),
    total_score SMALLINT DEFAULT 0,
    grade VARCHAR(2) NULL,                    -- A+, A, B, C, D, F (Subtask 2.5)
    scoring_version VARCHAR(10) NULL,         -- Track algorithm versions
    weights_used JSON NULL,                   -- Store weights used for calculation
    started_at TIMESTAMP NULL,
    completed_at TIMESTAMP NULL,
    error_message TEXT NULL,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    INDEX idx_domain_scans (domain_id, created_at DESC),
    INDEX idx_user_scans (user_id, created_at DESC),
    INDEX idx_scan_status (status) WHERE status IN ('pending', 'running')
);

CREATE TABLE scan_modules (
    id UUID PRIMARY KEY,
    scan_id UUID REFERENCES scans(id) ON DELETE CASCADE,
    module VARCHAR(20) NOT NULL,             -- security_headers|ssl_tls|dns_email
    score SMALLINT NULL,                     -- 0-100 module score
    status VARCHAR(10) NOT NULL,             -- ok|warn|fail|skipped|error
    raw JSON NULL,                           -- JSONB structured findings (Subtasks 2.2-2.4)
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    UNIQUE(scan_id, module),
    INDEX idx_module_status (module, status)
);

CREATE TABLE scoring_algorithms (
    id SERIAL PRIMARY KEY,
    version VARCHAR(10) UNIQUE,              -- v1.0, v1.1, etc. (Subtask 2.5)
    weights JSON,                            -- {ssl_tls: 0.40, security_headers: 0.30, dns_email: 0.30}
    caps JSON,                               -- Maximum deductions and limits
    grades JSON,                             -- Grade thresholds
    params JSON,                             -- Algorithm-specific parameters
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

## Security Scanning System (Subtasks 2.1-2.5)

### Scanner Architecture (Subtask 2.1)
```php
// app/Contracts/Scanners/Scanner.php
namespace App\Contracts\Scanners;
interface Scanner {
    public function scan(string $url, array $context = []): \App\Data\ModuleResult;
    public function name(): string; // security_headers|ssl_tls|dns_email
}

// app/Data/ModuleResult.php  
namespace App\Data;
class ModuleResult {
    public function __construct(
        public string $module,
        public int $score,            // 0..100
        public string $status,        // ok|warn|fail|skipped|error
        public array $raw = [],       // structured findings for JSONB storage
    ) {}
}
```

### Scanner Implementations

#### SecurityHeadersScanner (Subtask 2.3 - 30% weight)
```php
class SecurityHeadersScanner implements Scanner {
    public function name(): string { return 'security_headers'; }
    
    public function scan(string $url, array $context = []): ModuleResult {
        // Analyzes critical security headers with weighted scoring:
        // - HSTS: 35 points (requires 180+ days max-age, includeSubDomains, preload)
        // - CSP: 35 points (deductions for unsafe-inline, unsafe-eval)
        // - X-Frame-Options: 10 points (DENY or SAMEORIGIN)
        // - X-Content-Type-Options: 10 points (nosniff)
        // - Referrer-Policy: 10 points (strict policies)
        
        // Returns structured findings in raw array for JSONB storage
    }
}
```

#### SslTlsScanner (Subtask 2.2 - 40% weight)
```php
class SslTlsScanner implements Scanner {
    public function name(): string { return 'ssl_tls'; }
    
    public function scan(string $url, array $context = []): ModuleResult {
        // Comprehensive SSL/TLS analysis:
        // - Certificate validity and expiration (100% deduction if expired)
        // - Protocol versions (25% deduction for TLSv1.0/1.1)
        // - Cipher strength (20% deduction for weak ciphers)
        // - Perfect Forward Secrecy validation
        // - TLS 1.3 bonus scoring
        
        // Uses stream_socket_client with SSL context for deep inspection
    }
}
```

#### DnsEmailScanner (Subtask 2.4 - 30% weight)
```php
class DnsEmailScanner implements Scanner {
    public function name(): string { return 'dns_email'; }
    
    public function scan(string $url, array $context = []): ModuleResult {
        // DNS and email security verification:
        // - SPF records (15 points deduction if missing/permissive)
        // - DKIM validation with custom selector support
        // - DMARC policy checking (30 points for none, 20 for quarantine, 0 for reject)
        // - DNSSEC validation (10 points if missing)
        // - CAA records (5 points if missing)
        
        // Supports email_mode: 'expected' vs 'none' for non-email domains
        // Custom dkim_selector from domain configuration
    }
}
```

### SSRF Protection (Subtask 2.1)
```php
// app/Support/NetworkGuard.php
final class NetworkGuard {
    private const BLOCKED_NETWORKS = [
        '10.0.0.0/8',      // Private networks
        '172.16.0.0/12',   // Private networks  
        '192.168.0.0/16',  // Private networks
        '127.0.0.0/8',     // Localhost
        'fd00::/8'         // IPv6 private
    ];
    
    public static function assertPublicHttpUrl(string $url): array {
        // Validates URL is public and safe to scan
        // Resolves DNS to verify all IPs are public
        // Prevents SSRF attacks against internal infrastructure
        // Throws InvalidArgumentException for blocked URLs
    }
    
    public static function followRedirectsSafely(string $url, int $max = 3): string {
        // Safely follows HTTP redirects while validating each hop
        // Ensures all redirect targets are also public
        // Prevents redirect-based SSRF attacks
    }
}
```

### Scoring Engine (Subtask 2.5)
```php
// app/Services/Scoring/SecurityScorer.php
final class SecurityScorer {
    public function __construct(private array $config) {}
    
    public function calculate(array $subscores): array {
        // Weighted scoring calculation:
        // - SSL/TLS: 40% weight (most critical for security)
        // - Security Headers: 30% weight (important for modern security)  
        // - DNS/Email: 30% weight (foundation security)
        
        // Normalizes weights if not all scanners run
        // Returns total score (0-100), grade (A+ to F), and weights used
        // Stores scoring_version for algorithm tracking
    }
    
    private function grade(int $score): string {
        // Grade mapping from product.md competitive analysis:
        // A+: 97-100, A: 90-96, B: 80-89, C: 70-79, D: 60-69, F: 0-59
    }
}
```

## Laravel Cloud Queue Architecture (Subtasks 3.1-3.3)

### Auto-scaling Queue Clusters (Subtask 3.1)
**Revolutionary Improvement**: Laravel Cloud Queue clusters eliminate manual Horizon configuration.

```php
// OLD APPROACH: Manual Horizon Configuration (NOT USED)
'environments' => [
    'production' => [
        'default' => [
            'connection' => 'redis',
            'queue' => ['default'],
            'balance' => 'auto',
            'minProcesses' => 1,
            'maxProcesses' => 10,
            'tries' => 3,
            'timeout' => 300,
        ],
    ],
]

// NEW APPROACH: Auto-scaling Queue Clusters (Subtask 3.1)
// Zero manual configuration required - Laravel Cloud automatically manages:
// - Worker process scaling based on queue latency thresholds
// - CPU and RAM monitoring per worker process
// - Job throughput optimization and backlog management
// - Automatic cost reduction when queue is idle
// - Real-time metrics and performance monitoring
```

### Queue Job Implementation (Subtask 3.2)
```php
// app/Jobs/RunDomainScan.php
class RunDomainScan implements ShouldQueue {
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    public function __construct(private Scan $scan) {}
    
    public function handle(
        SecurityHeadersScanner $headersScanner,
        SslTlsScanner $sslScanner,
        DnsEmailScanner $dnsScanner,
        SecurityScorer $scorer
    ): void {
        // Auto-scaling features leveraged:
        // - Workers scale up when scan jobs queue during high demand
        // - Workers scale down when scan load decreases
        // - Real-time latency monitoring triggers scaling decisions
        // - Cost optimization through intelligent worker management
        
        // Orchestrates all three scanners and calculates final score
        // Stores results in JSONB format for efficient querying
        // Updates domain last_scan tracking for dashboard display
    }
    
    public function failed(Throwable $exception): void {
        // Handle scan failures with proper error logging
        // Update scan status to 'failed' with error message
    }
}
```

### HTTP Service Provider (Subtask 3.3)
```php
// app/Providers/ScannerServiceProvider.php
public function boot(): void {
    \Http::macro('safe', function () {
        return \Http::withHeaders(['User-Agent' => config('scanner.user_agent')])
            ->timeout(config('scanner.timeout'))
            ->connectTimeout(config('scanner.connect_timeout'))
            ->withOptions([
                'verify' => true,           // SSL verification required
                'allow_redirects' => false  // Manual redirect handling for security
            ]);
    });
}
```

## Laravel Cloud Infrastructure (Subtasks 8.1-8.3)

### Serverless Database Configuration (Subtask 8.1)
```php
// Auto-scaling PostgreSQL with hibernation cost optimization
'database' => [
    'type' => 'serverless-postgres',
    'version' => 15,
    'auto_scaling' => true,
    'hibernation' => true,              // Cost optimization during low usage
    'compute_units' => [
        'min' => 0.5,                   // Hibernates to near-zero cost
        'max' => 4.0,                   // Scales based on query demand
    ],
    'storage' => 'auto-scaling',        // Grows automatically with data
    'backup_retention' => 30            // 30-day backup retention
]
```

### Caching Strategy (Auto-managed Redis)
```php
// Auto-scaling Redis Cache configuration
'cache_layers' => [
    // Scanner result caching for repeat domain scans
    'scan_results:{domain}:{module}:{hash}' => 3600,     // 1 hour TTL
    
    // Dashboard metrics caching for performance
    'dashboard:{user_id}'                   => 900,      // 15 minutes TTL  
    'user_domains:{user_id}'                => 1800,     // 30 minutes TTL
    
    // Report caching for S3 storage optimization  
    'pdf_report:{scan_id}'                  => 604800,   // 7 days TTL
    
    // Trend analysis caching for charts
    'security_trends:{user_id}:{period}'    => 3600,     // 1 hour TTL
]
```

### Environment Configuration (Subtask 8.2)
```yaml
# Laravel Cloud Auto-Configuration
provider: aws
region: us-east-2  # Ohio for optimal latency
type: web
size: small        # Auto-scales based on traffic

# Auto-managed Services
database:
  type: serverless-postgres
  version: 15
  auto_scaling: true
  hibernation: true

cache:
  type: redis
  auto_scaling: true

storage:
  type: s3
  bucket: achilleus-reports

# Queue Clusters (Developer Preview)
queue_clusters:
  type: auto-scaling
  latency_threshold: configurable  # Triggers worker scaling
  min_workers: 0                   # Scales to zero when idle
  max_workers: auto                # No upper limit, demand-based
  
# Application Auto-scaling
autoscaling:
  min_replicas: 1
  max_replicas: 10
  target_cpu: 70
  hibernation:
    enabled: true
    after_minutes: 30              # Hibernates after 30 minutes idle
```

## Scanner Configuration (Subtasks 2.1, 2.5)

### Scanner Settings (Subtask 2.1)
```php
// config/scanner.php
return [
    'user_agent' => 'AchilleusScanner/1.0 (+https://achilleus.io)',
    'timeout' => 15,                    // HTTP request timeout
    'connect_timeout' => 5,             // Connection establishment timeout
    'max_redirects' => 3,               // Maximum redirect following
    'blocked_ips' => [
        '127.0.0.1',                    // Localhost IPv4
        '0.0.0.0',                      // Invalid/any address
        '::1',                          // Localhost IPv6  
        '169.254.169.254'               // AWS metadata service
    ],
    'blocked_cidrs' => [
        '10.0.0.0/8',                   // Private Class A
        '172.16.0.0/12',                // Private Class B
        '192.168.0.0/16',               // Private Class C
        'fc00::/7'                      // IPv6 private networks
    ],
];
```

### Scoring Configuration (Subtask 2.5)
```php
// config/scoring.php
return [
    'version' => 'v1.0',               // Algorithm version tracking
    'weights' => [
        'security_headers' => 0.30,    // 30% - Modern web security
        'ssl_tls' => 0.40,            // 40% - Critical foundation security  
        'dns_email' => 0.30,          // 30% - Network and email security
    ],
    'caps' => [
        'no_ssl_on_https_final_cap' => 15,  // Maximum deduction for specific issues
    ],
    'grades' => [
        ['min' => 97, 'grade' => 'A+'], // Excellent security posture
        ['min' => 90, 'grade' => 'A'],  // Strong security implementation
        ['min' => 80, 'grade' => 'B'],  // Good security with minor issues
        ['min' => 70, 'grade' => 'C'],  // Adequate security, needs improvement
        ['min' => 60, 'grade' => 'D'],  // Poor security, significant issues
        ['min' => 0,  'grade' => 'F'],  // Failing security, critical problems
    ],
    'params' => [
        'security_headers' => [
            'hsts_min' => 15552000,     // 180 days minimum for HSTS max-age
        ],
    ],
];
```

## API Architecture (Subtasks 4.2-4.3, 7.1-7.6)

### Inertia Routes (Server-Side Routing)
```php
// routes/web.php - Not REST API, Inertia page routing
Route::middleware(['auth', 'verified'])->group(function () {
    // Dashboard and core pages (Subtask 7.2)
    Route::get('/dashboard', [DashboardController::class, 'index']);
    
    // Domain management (Subtasks 4.2, 7.3)
    Route::get('/domains', [DomainController::class, 'index']);
    Route::post('/domains', [DomainController::class, 'store']);
    Route::put('/domains/{domain}', [DomainController::class, 'update']);
    Route::delete('/domains/{domain}', [DomainController::class, 'destroy']);
    
    // Scan operations (Subtasks 4.3, 7.4)
    Route::post('/domains/{domain}/scan', [DomainScanController::class, 'store']);
    Route::get('/scans/{scan}', [ScanController::class, 'show']);
    Route::get('/activity', [ActivityController::class, 'index']);
    
    // Reporting system (Subtasks 6.1-6.2, 7.6)
    Route::get('/reports', [ReportController::class, 'index']);
    Route::post('/reports', [ReportController::class, 'store']);
    
    // Billing integration (Subtasks 5.1-5.3, 7.5)
    Route::get('/billing', [BillingController::class, 'index']);
});
```

### Controller Implementation Examples

#### Scan Controller (Subtask 4.3)
```php
// app/Http/Controllers/DomainScanController.php
class DomainScanController extends Controller {
    public function store(Domain $domain, SecurityScanService $scanner) {
        $this->authorize('scan', $domain);
        
        // Rate limiting check (10 scans per minute per user)
        if ($domain->user->hasExceededScanLimit()) {
            return back()->with('error', 'Scan rate limit exceeded. Please wait.');
        }
        
        // Create scan record with pending status
        $scan = $domain->scans()->create([
            'user_id' => auth()->id(),
            'status' => 'pending',
            'scoring_version' => config('scoring.version'),
        ]);
        
        // Dispatch to auto-scaling queue clusters
        RunDomainScan::dispatch($scan);
        
        return redirect()->back()->with('success', 'Security scan initiated');
    }
}
```

### Authorization Implementation (Subtask 1.4)
```php
// app/Policies/DomainPolicy.php
class DomainPolicy {
    public function view(User $user, Domain $domain): bool {
        return $user->id === $domain->user_id;
    }
    
    public function scan(User $user, Domain $domain): bool {
        return $user->id === $domain->user_id 
            && $user->canScan()              // Check trial status and subscription
            && $domain->is_active            // Domain must be active
            && !$user->hasExceededScanLimit(); // Rate limiting check
    }
    
    public function update(User $user, Domain $domain): bool {
        return $user->id === $domain->user_id;
    }
    
    public function delete(User $user, Domain $domain): bool {
        return $user->id === $domain->user_id;
    }
}
```

## Payment Integration (Subtasks 5.1-5.3)

### Stripe Implementation (No Cashier - Subtask 5.1)
```php
// app/Services/StripeService.php
class StripeService {
    public function __construct(private \Stripe\StripeClient $stripe) {}
    
    public function createCustomer(User $user): \Stripe\Customer {
        return $this->stripe->customers->create([
            'email' => $user->email,
            'metadata' => ['user_id' => $user->id],
            'name' => $user->name,
        ]);
    }
    
    public function createSubscription(User $user, string $paymentMethodId): \Stripe\Subscription {
        return $this->stripe->subscriptions->create([
            'customer' => $user->stripe_customer_id,
            'items' => [['price' => config('stripe.price_id')]],  // $27/month price ID
            'payment_behavior' => 'default_incomplete',
            'payment_settings' => [
                'save_default_payment_method' => 'on_subscription'
            ],
            'expand' => ['latest_invoice.payment_intent'],
            'trial_end' => $user->trial_ends_at->timestamp,      // Preserve 14-day trial
        ]);
    }
    
    public function updateSubscription(User $user, array $data): \Stripe\Subscription {
        return $this->stripe->subscriptions->update(
            $user->stripe_subscription_id,
            $data
        );
    }
}
```

### Billing Controllers (Subtask 5.2)
```php
// app/Http/Controllers/BillingController.php  
class BillingController extends Controller {
    public function index(StripeService $stripe) {
        $user = auth()->user();
        
        // Get subscription details if exists
        $subscription = $user->stripe_subscription_id 
            ? $stripe->getSubscription($user->stripe_subscription_id)
            : null;
            
        // Get payment methods
        $paymentMethods = $user->stripe_customer_id
            ? $stripe->getPaymentMethods($user->stripe_customer_id)
            : [];
            
        return Inertia::render('Billing/Index', [
            'subscription' => $subscription,
            'payment_methods' => $paymentMethods,
            'trial_days_left' => $user->trialDaysLeft(),
            'can_upgrade' => $user->isOnTrial(),
        ]);
    }
}
```

### Webhook Processing (Subtask 5.3)
```php
// app/Http/Controllers/StripeWebhookController.php
class StripeWebhookController extends Controller {
    public function handleWebhook(Request $request) {
        $payload = $request->getContent();
        $sig_header = $request->header('Stripe-Signature');
        
        try {
            $event = \Stripe\Webhook::constructEvent(
                $payload, 
                $sig_header, 
                config('stripe.webhook_secret')
            );
        } catch (\Exception $e) {
            return response('Invalid signature', 400);
        }
        
        match($event->type) {
            'customer.subscription.created' => $this->handleSubscriptionCreated($event),
            'customer.subscription.updated' => $this->handleSubscriptionUpdated($event),
            'customer.subscription.deleted' => $this->handleSubscriptionDeleted($event),
            'invoice.payment_failed' => $this->handlePaymentFailed($event),
            default => Log::info("Unhandled webhook: {$event->type}")
        };
        
        return response('Webhook handled', 200);
    }
}
```

## Reporting System (Subtasks 6.1-6.2)

### PDF Generation Engine (Subtask 6.1)
```php
// app/Services/ReportService.php
class ReportService {
    public function generatePdfReport(Scan $scan): string {
        $data = [
            'scan' => $scan,
            'domain' => $scan->domain,
            'user' => $scan->user,
            'modules' => $scan->modules()->get(),
            'generated_at' => now(),
            'company_logo' => asset('images/achilleus-logo.png'),
        ];
        
        $pdf = PDF::loadView('reports.security-scan', $data);
        $pdf->setPaper('A4', 'portrait');
        
        // Store in S3 with Laravel Cloud storage
        $filename = "reports/{$scan->id}-security-report.pdf";
        Storage::disk('s3')->put($filename, $pdf->output());
        
        return Storage::disk('s3')->url($filename);
    }
}
```

### Report Management (Subtask 6.2)
```php
// app/Models/Report.php
class Report extends Model {
    protected $fillable = [
        'scan_id',
        'user_id', 
        'filename',
        's3_path',
        'file_size',
        'generated_at',
    ];
    
    public function scan(): BelongsTo {
        return $this->belongsTo(Scan::class);
    }
    
    public function user(): BelongsTo {
        return $this->belongsTo(User::class);
    }
    
    public function getDownloadUrlAttribute(): string {
        return Storage::disk('s3')->temporaryUrl($this->s3_path, now()->addMinutes(10));
    }
}
```

## File Structure (Complete Implementation Reference)

### Complete Scanner Architecture
```
app/
├── Contracts/Scanners/
│   └── Scanner.php                 # Scanner interface (Subtask 2.1)
├── Data/
│   └── ModuleResult.php            # Scan result data structure (Subtask 2.1)
├── Http/Controllers/
│   ├── DomainController.php        # Domain CRUD (Subtask 4.2)
│   ├── DomainScanController.php    # Scan triggering (Subtask 4.3)
│   ├── BillingController.php       # Stripe billing (Subtask 5.2)
│   └── StripeWebhookController.php # Webhook handling (Subtask 5.3)
├── Jobs/
│   └── RunDomainScan.php          # Auto-scaling scan job (Subtask 3.2)
├── Models/
│   ├── User.php                   # Enhanced with Stripe methods (Subtask 1.4)
│   ├── Domain.php                 # Domain management (Subtask 4.1)
│   ├── Scan.php                   # Scan tracking (Subtask 1.2)
│   └── Report.php                 # PDF report management (Subtask 6.2)
├── Policies/
│   └── DomainPolicy.php           # Domain authorization (Subtask 1.4)
├── Providers/
│   └── ScannerServiceProvider.php # HTTP macro registration (Subtask 3.3)
├── Services/
│   ├── Scanners/
│   │   ├── SecurityHeadersScanner.php # HSTS, CSP analysis (Subtask 2.3)
│   │   ├── SslTlsScanner.php          # Certificate analysis (Subtask 2.2)
│   │   └── DnsEmailScanner.php        # SPF, DKIM, DMARC (Subtask 2.4)
│   ├── Scoring/
│   │   └── SecurityScorer.php         # Weighted scoring (Subtask 2.5)
│   ├── StripeService.php              # Payment processing (Subtask 5.1)
│   └── ReportService.php              # PDF generation (Subtask 6.1)
└── Support/
    └── NetworkGuard.php               # SSRF protection (Subtask 2.1)

config/
├── scanner.php                        # Scanner configuration (Subtask 2.1)
└── scoring.php                        # Scoring weights (Subtask 2.5)

database/migrations/
└── 2025_01_01_000001_create_scans_tables.php # Complete schema (Subtask 1.2)

resources/js/Pages/                    # React components (Subtasks 7.1-7.6)
├── Auth/
│   ├── Login.tsx                      # Custom auth UI (Subtask 7.1)
│   ├── Register.tsx
│   └── ForgotPassword.tsx
├── Dashboard/
│   └── Index.tsx                      # 4-card dashboard (Subtask 7.2)
├── Domains/
│   ├── Index.tsx                      # Domain management (Subtask 7.3)
│   ├── Create.tsx
│   └── Edit.tsx
├── Scans/
│   └── Show.tsx                       # Scan results display (Subtask 7.4)
├── Billing/
│   └── Index.tsx                      # Billing interface (Subtask 7.5)
└── Reports/
    └── Index.tsx                      # Report management (Subtask 7.6)

routes/
└── web.php                            # Inertia routes (Subtasks 4.2-4.3, 7.1-7.6)
```

This technical documentation now provides complete implementation guidance mapped to the execution.md subtasks, ensuring developers can build the MVP systematically while understanding the architectural decisions and Laravel Cloud optimizations.