# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Achilleus** - Security monitoring SaaS for developers managing multiple websites
**Target**: Freelancers and small businesses needing affordable security monitoring
**Price**: $27/month for 10 domains with unlimited scans
**Stack**: Laravel 12 + React 19 + Shadcn/ui + Inertia.js + Playwright MCP

## Documentation Structure

Complete project documentation is available in `/docs/`:
- `/docs/technical.md` - System architecture, database schema, scanner implementation
- `/docs/product.md` - Product specifications, UI mockups, business rules  
- `/docs/execution.md` - Development phases, testing strategy, deployment plan

## Achilleus Development Shortcuts

### Development Workflow Commands

```bash
# Achilleus Plan 1.1 - Plan your task
achilleus-plan() {
    echo "📋 PLANNING PHASE"
    echo "1. Break down task into testable units"
    echo "2. Identify security implications"
    echo "3. Plan test cases (unit + integration + E2E)"
    echo "4. Consider SSRF/validation requirements"
}

# Achilleus Test 1.1 - Write tests first (TDD)
achilleus-test() {
    echo "🧪 TESTING PHASE - Write tests FIRST"
    echo "Unit Tests: php artisan pest tests/Unit/"
    echo "Feature Tests: php artisan pest tests/Feature/"
    echo "E2E Tests: npx playwright test"
    echo "Security Tests: Focus on input validation & SSRF"
}

# Achilleus Code 1.1 - Implement to pass tests
achilleus-code() {
    echo "💻 CODING PHASE - Make tests pass"
    echo "1. Follow Laravel conventions"
    echo "2. Use Shadcn/ui components"
    echo "3. Implement security-first patterns"
    echo "4. Add proper error handling"
}

# Achilleus Verify 1.1 - Test and validate
achilleus-verify() {
    echo "✅ VERIFICATION PHASE"
    echo "Running all tests..."
    php artisan pest --parallel
    npm run type-check
    npx playwright test
    php artisan pint --test
    echo "Manual verification commands:"
    echo "- php artisan tinker (test functions)"
    echo "- curl tests for APIs"
    echo "- Browser testing for UI"
}
```

Add these to your shell profile for quick access.

## Security-First Development (Critical for SaaS)

### 1. Input Validation & Sanitization
```php
// ALWAYS use Form Requests for validation
class StoreDomainRequest extends FormRequest {
    public function rules(): array {
        return [
            'url' => [
                'required',
                'url',
                'starts_with:https://', // HTTPS only
                new PublicUrl(),        // No private IPs
                new UniqueDomainForUser(auth()->id())
            ]
        ];
    }
    
    // Custom validation messages
    public function messages(): array {
        return [
            'url.starts_with' => 'Only HTTPS URLs are allowed for security.',
        ];
    }
}
```

### 2. SSRF Protection (Critical)
```php
// ALWAYS validate URLs before external requests
NetworkGuard::assertPublicHttpUrl($url);

// Blocked patterns:
// - localhost (127.0.0.1, ::1)
// - Private networks (10.x, 192.168.x, 172.16-31.x)
// - Cloud metadata (169.254.169.254)
// - Internal services (0.0.0.0)

// ENHANCED SCANNER PATTERN - Use AbstractScanner base class:
class SecurityHeadersScanner extends AbstractScanner {
    protected int $timeout = 20;           // Scanner-specific timeout
    protected int $retryAttempts = 2;      // Retry logic built-in
    
    public function name(): string {
        return 'security_headers';
    }
    
    public function getWeight(): int {
        return 30; // 30% of total score
    }
    
    protected function getRateLimitPerMinute(): int {
        return 8; // Polite to target servers
    }
    
    // Base class handles SSRF, retries, rate limiting, error handling
    protected function performScan(string $url, array $context = []): ModuleResult {
        // Your specific scanner logic here
        // NetworkGuard::assertPublicHttpUrl($url) called automatically
        // Timeout and retry logic handled by base class
    }
}
```

### 3. Authentication & Authorization
```php
// ALWAYS authorize domain access
class DomainController extends Controller {
    public function show(Domain $domain) {
        $this->authorize('view', $domain); // Check ownership
        return inertia('Domains/Show', compact('domain'));
    }
}

// Domain Policy
class DomainPolicy {
    public function view(User $user, Domain $domain): bool {
        return $user->id === $domain->user_id;
    }
}
```

### 4. Rate Limiting
```php
// Apply to all scan endpoints
Route::middleware(['throttle:scans'])->group(function () {
    Route::post('/domains/{domain}/scan', [DomainScanController::class, 'store']);
});

// In RouteServiceProvider
RateLimiter::for('scans', function (Request $request) {
    return Limit::perMinute(10)->by($request->user()->id);
});
```

### 5. Secure Data Handling
```php
// NEVER log sensitive data
Log::info('Scan started', [
    'domain_id' => $domain->id,
    'user_id' => $user->id,
    // NEVER log: URLs, API keys, personal data
]);

// Use encrypted casting for sensitive fields
class User extends Model {
    protected $casts = [
        'stripe_customer_id' => 'encrypted', // If storing sensitive IDs
    ];
}
```

### 6. Enhanced Scanner Implementation (Critical)

#### Scanner Architecture
```php
// ALWAYS extend AbstractScanner for consistency
abstract class AbstractScanner implements Scanner {
    protected int $timeout = 30;           // Default timeout
    protected int $retryAttempts = 3;      // Retry with exponential backoff
    protected int $retryDelay = 1;         // Base delay in seconds
    
    // Template method - handles all common concerns
    public function scan(string $url, array $context = []): ModuleResult {
        // ✅ Automatic rate limiting check
        // ✅ SSRF protection via NetworkGuard
        // ✅ Retry logic with exponential backoff
        // ✅ Consistent error handling and logging
        // ✅ Execution time tracking
        
        return $this->performScan($url, $context);
    }
    
    // Implement this method in concrete scanners
    abstract protected function performScan(string $url, array $context = []): ModuleResult;
}
```

#### Scanner Error Handling Hierarchy
```php
// Use specific exceptions for different failure types
namespace App\Exceptions;

class ScannerException extends Exception {}      // Base scanner exception
class NetworkException extends ScannerException {} // Network/connectivity issues
class TimeoutException extends ScannerException {} // Request timeouts
class RateLimitException extends ScannerException {} // Rate limit hit
class ValidationException extends ScannerException {} // Data validation
class SSRFException extends ValidationException {}  // SSRF attempt blocked

// Handle each exception type appropriately
try {
    $result = $scanner->scan($url);
} catch (SSRFException $e) {
    // Log security incident, return error result
    Log::warning('SSRF blocked', ['url' => $url]);
    return ModuleResult::error($scanner->name(), 'Invalid URL');
} catch (TimeoutException $e) {
    // Different handling for timeouts
    return ModuleResult::timeout($scanner->name(), $e->getExecutionTime());
}
```

#### Rate Limiting Best Practices
```php
class SslTlsScanner extends AbstractScanner {
    protected function getRateLimitPerMinute(): int {
        return 5; // Conservative for SSL connections
    }
}

class SecurityHeadersScanner extends AbstractScanner {
    protected function getRateLimitPerMinute(): int {
        return 8; // Moderate for HTTP requests
    }
}

class DnsEmailScanner extends AbstractScanner {
    protected function getRateLimitPerMinute(): int {
        return 15; // Higher for DNS queries (lighter load)
    }
}

// Rate limiting implementation respects target servers
private function checkRateLimit(string $url): bool {
    $host = parse_url($url, PHP_URL_HOST);
    $key = "scanner_rate_limit:{$this->name()}:{$host}";
    
    $requests = Cache::get($key, 0);
    return $requests < $this->getRateLimitPerMinute();
}
```

#### Enhanced SSL Chain Validation
```php
class SslTlsScanner extends AbstractScanner {
    protected function performScan(string $url, array $context = []): ModuleResult {
        // Enhanced SSL context with comprehensive security
        $context = stream_context_create([
            'ssl' => [
                'capture_peer_cert' => true,
                'capture_peer_cert_chain' => true,  // Full chain analysis
                'verify_peer' => true,
                'verify_peer_name' => true,
                'allow_self_signed' => false,
                'disable_compression' => true,      // Prevent CRIME attacks
                'crypto_method' => STREAM_CRYPTO_METHOD_TLSv1_2_CLIENT | STREAM_CRYPTO_METHOD_TLSv1_3_CLIENT
            ]
        ]);
        
        // Comprehensive certificate analysis
        $analysis = $this->performEnhancedAnalysis($cert, $chain, $meta, $host);
        
        return new ModuleResult(
            module: $this->name(),
            score: $analysis['score'],
            status: $this->determineStatus($analysis['score']),
            raw: $analysis['details']
        );
    }
    
    private function performEnhancedAnalysis($cert, $chain, $meta, $host): array {
        // ✅ Expiration checking with tiered warnings
        // ✅ Hostname verification (CN + SAN)
        // ✅ Protocol version analysis (TLS 1.2+)
        // ✅ Cipher strength evaluation
        // ✅ Perfect Forward Secrecy check
        // ✅ Key size and type validation
        // ✅ Certificate chain validation
        // ✅ Signature algorithm strength
        // ✅ Wildcard certificate handling
    }
}
```

#### Security Headers Deep Analysis
```php
class SecurityHeadersScanner extends AbstractScanner {
    protected function performScan(string $url, array $context = []): ModuleResult {
        // Secure HTTP client configuration
        $response = Http::timeout($this->timeout)
            ->withUserAgent('Achilleus Security Scanner/1.0')
            ->withOptions([
                'verify' => true,                    // Verify SSL certificates
                'allow_redirects' => [
                    'max' => 3,                      // Limit redirects
                    'strict' => true,
                    'referer' => false,
                    'protocols' => ['https']         // HTTPS only
                ]
            ])
            ->get($url);
            
        // Comprehensive header analysis
        $analysis = $this->performHeaderAnalysis($headers, $url);
        // ✅ HSTS analysis (max-age, includeSubDomains, preload)
        // ✅ CSP analysis (unsafe directives, important directives)
        // ✅ X-Content-Type-Options verification
        // ✅ X-Frame-Options checking
        // ✅ Referrer-Policy evaluation
        // ✅ Additional security headers (Permissions-Policy, etc.)
        // ✅ Information disclosure detection (Server headers)
    }
}
```

#### DNS/Email Comprehensive Analysis
```php
class DnsEmailScanner extends AbstractScanner {
    protected function performScan(string $url, array $context = []): ModuleResult {
        $host = parse_url($url, PHP_URL_HOST);
        $emailMode = $context['email_mode'] ?? 'expected';
        
        // Timeout protection for DNS queries
        $oldTimeout = ini_get('default_socket_timeout');
        ini_set('default_socket_timeout', $this->timeout);
        
        try {
            $analysis = $this->performDnsAnalysis($host, $emailMode);
            // ✅ Basic DNS connectivity (A/AAAA records)
            // ✅ MX record validation
            // ✅ SPF policy analysis (permissive detection)
            // ✅ DKIM validation (common selector fallback)
            // ✅ DMARC policy evaluation (reject > quarantine > none)
            // ✅ CAA record checking
            // ✅ DNSSEC validation (when available)
            
        } finally {
            ini_set('default_socket_timeout', $oldTimeout);
        }
    }
}
```

## Laravel Best Practices

### 1. Service Layer Pattern
```php
// Controllers should be thin
class DomainController extends Controller {
    public function store(StoreDomainRequest $request, DomainService $service) {
        $domain = $service->createDomain($request->validated());
        return redirect()->route('domains.index');
    }
}

// Business logic in services
class DomainService {
    public function createDomain(array $data): Domain {
        $data['url'] = $this->normalizeUrl($data['url']);
        $domain = Domain::create($data);
        $this->triggerInitialScan($domain);
        return $domain;
    }
}
```

### 2. Queue Job Pattern
```php
// Jobs should be idempotent and recoverable
class RunDomainScan implements ShouldQueue {
    public $tries = 3;
    public $timeout = 60;
    public $backoff = [10, 30, 60]; // Exponential backoff
    
    public function handle() {
        // Update status first
        $this->scan->update(['status' => 'running']);
        
        try {
            // Main logic
            $this->executeScanners();
        } catch (Exception $e) {
            // Clean up on failure
            $this->scan->update(['status' => 'failed', 'error_message' => $e->getMessage()]);
            throw $e;
        }
    }
    
    public function failed(Throwable $exception): void {
        // Always implement failed handler
        $this->scan->update(['status' => 'failed']);
    }
}
```

### 3. Model Relationships & Scopes
```php
class Domain extends Model {
    // Define relationships clearly
    public function user(): BelongsTo {
        return $this->belongsTo(User::class);
    }
    
    public function scans(): HasMany {
        return $this->hasMany(Scan::class)->orderBy('created_at', 'desc');
    }
    
    // Useful scopes
    public function scopeActive(Builder $query): void {
        $query->where('is_active', true);
    }
    
    // Computed attributes
    public function getLastScanAttribute(): ?Scan {
        return $this->scans()->first();
    }
}
```

## UX Enhancement Best Practices (Critical for Conversion)

### 1. Context-Aware Trial Conversion
```typescript
// ALWAYS track user progress and show relevant conversion messages
interface TrialProgress {
  daysRemaining: number
  domainsAdded: number
  scansCompleted: number
  reportsGenerated: number
  completionPercentage: number
}

// Smart messaging based on user behavior
const getTrialMessage = (progress: TrialProgress) => {
  if (progress.daysRemaining <= 3 && progress.scansCompleted === 0) {
    return {
      urgency: 'high',
      message: 'Trial expires soon - Run your first scan now!',
      cta: 'Start Free Scan'
    }
  }
  
  if (progress.completionPercentage >= 80) {
    return {
      urgency: 'medium', 
      message: `You've explored ${progress.completionPercentage}% of Achilleus!`,
      cta: 'Upgrade Now'
    }
  }
  
  // Default progression message
  return {
    urgency: 'low',
    message: `${progress.daysRemaining} days left - ${10 - progress.domainsAdded} slots available`,
    cta: 'Add Domain'
  }
}
```

### 2. Real-Time Scan Progress (Essential for UX)
```typescript
// ALWAYS show progress for long-running operations
export function ScanProgress({ scanId }: { scanId: string }) {
  const [steps, setSteps] = useState([
    { name: 'SSL/TLS Analysis', status: 'pending', progress: 0 },
    { name: 'Security Headers', status: 'pending', progress: 0 },
    { name: 'DNS/Email Security', status: 'pending', progress: 0 }
  ])

  useEffect(() => {
    // WebSocket connection for real-time updates
    const channel = Echo.channel(`scan.${scanId}`)
      .listen('ScanStepStarted', (e) => {
        setSteps(prev => prev.map((step, i) => 
          i === e.stepIndex 
            ? { ...step, status: 'running', message: 'Analyzing...' }
            : step
        ))
      })
      .listen('ScanStepCompleted', (e) => {
        setSteps(prev => prev.map((step, i) => 
          i === e.stepIndex 
            ? { ...step, status: 'completed', progress: 100 }
            : step
        ))
      })
      
    return () => channel.leave()
  }, [scanId])

  // Show visual progress for each step
  return (
    <div className="space-y-3">
      {steps.map((step, index) => (
        <div key={step.name} className="flex items-center gap-3">
          <StatusIndicator status={step.status} />
          <div className="flex-1">
            <div className="flex justify-between">
              <span>{step.name}</span>
              {step.status === 'running' && (
                <Badge>{step.progress}%</Badge>
              )}
            </div>
            {step.status === 'running' && (
              <Progress value={step.progress} className="h-1 mt-1" />
            )}
          </div>
        </div>
      ))}
    </div>
  )
}
```

### 3. Smart Empty States (Drive Engagement)
```typescript
// NEVER show empty tables/lists without actionable guidance
export function EmptyState({ type, user }: { type: string, user: User }) {
  const isTrialing = user.subscription_status === 'trialing'
  const daysLeft = dayjs(user.trial_ends_at).diff(dayjs(), 'day')
  
  const getEmptyConfig = () => {
    switch (type) {
      case 'domains':
        return {
          title: isTrialing ? 'Add Your First Domain' : 'No Domains Yet',
          description: isTrialing 
            ? `${daysLeft} days left in trial - Add a domain to see Achilleus in action!`
            : 'Start monitoring your website security',
          primaryAction: 'Add Domain',
          secondaryAction: isTrialing ? 'Try Demo Domain' : 'View Documentation',
          tips: [
            'Add up to 10 domains',
            'HTTPS domains only',
            'Unlimited scans included'
          ]
        }
        
      case 'scans':
        return {
          title: 'No Scans Yet',
          description: 'Run your first security scan to get detailed analysis',
          primaryAction: 'Start First Scan',
          secondaryAction: 'Add Another Domain',
          tips: [
            'Scans complete in 30 seconds',
            'Detailed security recommendations', 
            'Track improvements over time'
          ]
        }
    }
  }

  // Always show clear next steps with value props
  return (
    <div className="text-center py-12">
      <Icon className="mx-auto mb-4" />
      <h3 className="text-lg font-semibold mb-2">{config.title}</h3>
      <p className="text-muted-foreground mb-6">{config.description}</p>
      
      <div className="flex gap-3 justify-center mb-6">
        <Button onClick={handlePrimaryAction}>{config.primaryAction}</Button>
        <Button variant="outline" onClick={handleSecondaryAction}>
          {config.secondaryAction}
        </Button>
      </div>
      
      {/* Always show value props */}
      <div className="max-w-md mx-auto">
        <p className="text-sm font-medium mb-2">What you get:</p>
        <ul className="text-sm text-muted-foreground">
          {config.tips.map(tip => (
            <li key={tip} className="flex items-center gap-2">
              <Check className="h-3 w-3 text-green-500" />
              {tip}
            </li>
          ))}
        </ul>
      </div>
    </div>
  )
}
```

### 4. Security Score Tooltips (Educational)
```typescript
// ALWAYS explain what scores mean - educate users
export function ScoreTooltip({ score, grade, module, children }) {
  const getExplanation = () => {
    const gradeDescriptions = {
      'A+': 'Excellent security - industry leading',
      'A': 'Very good security configuration',
      'B+': 'Good security with minor improvements',
      'B': 'Acceptable security, room for improvement',
      'C': 'Significant security gaps need attention',
      'D': 'Poor security - immediate action required',
      'F': 'Critical security failures'
    }
    
    const moduleWeights = {
      'ssl_tls': { weight: '40%', title: 'SSL/TLS Security' },
      'security_headers': { weight: '30%', title: 'Security Headers' },
      'dns_email': { weight: '30%', title: 'DNS & Email Security' }
    }
    
    return {
      grade: gradeDescriptions[grade],
      module: module ? moduleWeights[module] : null
    }
  }

  return (
    <Tooltip>
      <TooltipTrigger asChild>{children}</TooltipTrigger>
      <TooltipContent className="max-w-sm p-4">
        <div className="space-y-2">
          <div className="flex items-center gap-2">
            <Badge variant="outline" className={getGradeColor(grade)}>
              {grade}
            </Badge>
            <span className="font-semibold">{score}/100</span>
          </div>
          
          <p className="text-sm">{explanation.grade}</p>
          
          {explanation.module && (
            <div>
              <p className="text-sm font-medium">
                {explanation.module.title} ({explanation.module.weight} of total)
              </p>
            </div>
          )}
          
          <p className="text-xs text-muted-foreground pt-2 border-t">
            Scores based on industry security standards and best practices
          </p>
        </div>
      </TooltipContent>
    </Tooltip>
  )
}
```

### 5. Prominent Quick Actions (Reduce Friction)
```typescript
// ALWAYS provide clear next actions in every context
export function QuickActions({ context, user, data }) {
  const getActions = () => {
    switch (context) {
      case 'dashboard':
        return {
          primary: data?.domains?.length > 0 
            ? { label: 'Scan All Domains', icon: Search, action: handleScanAll }
            : { label: 'Add First Domain', icon: Plus, action: handleAddDomain },
          secondary: [
            { 
              label: 'Generate Report', 
              icon: FileText, 
              disabled: !data?.hasScans,
              action: handleGenerateReport 
            },
            { 
              label: 'View Activity', 
              icon: Clock, 
              badge: data?.recentScansCount,
              action: () => router.visit('/activity')
            }
          ]
        }
        
      case 'domains':
        return {
          primary: { label: 'Add Domain', icon: Plus, action: handleAddDomain },
          secondary: [
            { 
              label: 'Scan All', 
              icon: Search, 
              disabled: !data?.domains?.length,
              action: handleScanAll 
            },
            { 
              label: 'Export List', 
              icon: Download, 
              disabled: !data?.domains?.length,
              action: handleExport 
            }
          ]
        }
        
      case 'activity':
        return {
          primary: { label: 'New Scan', icon: Search, action: handleNewScan },
          secondary: [
            { 
              label: 'Generate Report', 
              icon: FileText, 
              disabled: !data?.scans?.length,
              action: handleGenerateReport 
            },
            { 
              label: 'Download CSV', 
              icon: Download, 
              disabled: !data?.scans?.length,
              action: handleDownloadCsv 
            }
          ]
        }
    }
  }

  const actions = getActions()
  
  // Always show primary action prominently
  return (
    <div className="flex flex-col sm:flex-row gap-3">
      <Button 
        size="default"
        onClick={actions.primary.action}
        disabled={actions.primary.loading}
        className="min-w-[140px]"
      >
        <actions.primary.icon className="h-4 w-4 mr-2" />
        {actions.primary.label}
      </Button>
      
      <div className="flex gap-2">
        {actions.secondary.map((action, index) => (
          <Button
            key={index}
            variant="outline"
            onClick={action.action}
            disabled={action.disabled}
          >
            <action.icon className="h-4 w-4 mr-2" />
            {action.label}
            {action.badge && (
              <Badge variant="secondary" className="ml-2">
                {action.badge}
              </Badge>
            )}
          </Button>
        ))}
      </div>
    </div>
  )
}
```

### 6. User Progress Tracking (Analytics)
```typescript
// Track user engagement for conversion optimization
export function useTrialProgress(user: User) {
  const [progress, setProgress] = useState<TrialProgress>({
    daysRemaining: 0,
    domainsAdded: 0,
    scansCompleted: 0,
    reportsGenerated: 0,
    completionPercentage: 0
  })

  useEffect(() => {
    const calculateProgress = () => {
      const daysRemaining = dayjs(user.trial_ends_at).diff(dayjs(), 'day')
      const domainsAdded = user.domains?.length || 0
      const scansCompleted = user.scans?.length || 0
      const reportsGenerated = user.reports?.length || 0
      
      // Weight different actions for completion percentage
      const weights = {
        domain: 25,    // Adding first domain (25%)
        scan: 40,      // Running scans (40%) 
        report: 25,    // Generating reports (25%)
        billing: 10    // Viewing billing (10%)
      }
      
      let completionPercentage = 0
      if (domainsAdded > 0) completionPercentage += weights.domain
      if (scansCompleted > 0) completionPercentage += weights.scan
      if (reportsGenerated > 0) completionPercentage += weights.report
      if (user.viewed_billing) completionPercentage += weights.billing
      
      setProgress({
        daysRemaining,
        domainsAdded,
        scansCompleted, 
        reportsGenerated,
        completionPercentage
      })
    }
    
    calculateProgress()
  }, [user])

  // Track conversion events
  const trackEvent = (event: string, metadata?: any) => {
    // Send to analytics service
    analytics.track(`trial_${event}`, {
      user_id: user.id,
      days_remaining: progress.daysRemaining,
      completion_percentage: progress.completionPercentage,
      ...metadata
    })
  }

  return { progress, trackEvent }
}
```

## React + Shadcn/ui Best Practices

### 1. Component Structure
```typescript
// Use Shadcn/ui components consistently
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/Components/ui/card"
import { Badge } from "@/Components/ui/badge"
import { Button } from "@/Components/ui/button"

interface DashboardCardProps {
    title: string
    value: string | number
    description?: string
    variant?: "default" | "destructive" | "outline" | "secondary"
}

export function DashboardCard({ title, value, description, variant = "default" }: DashboardCardProps) {
    return (
        <Card>
            <CardHeader className="pb-2">
                <CardTitle className="text-sm font-medium">{title}</CardTitle>
            </CardHeader>
            <CardContent>
                <div className="text-2xl font-bold">{value}</div>
                {description && (
                    <CardDescription className="text-xs text-muted-foreground">
                        {description}
                    </CardDescription>
                )}
            </CardContent>
        </Card>
    )
}
```

### 2. Form Handling with React Hook Form + Zod
```typescript
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import * as z from "zod"
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from "@/Components/ui/form"

const domainSchema = z.object({
    url: z.string()
        .url("Please enter a valid URL")
        .refine(url => url.startsWith('https://'), "Only HTTPS URLs are allowed"),
    email_mode: z.enum(["expected", "none"])
})

export function AddDomainForm() {
    const form = useForm<z.infer<typeof domainSchema>>({
        resolver: zodResolver(domainSchema),
        defaultValues: { email_mode: "expected" }
    })
    
    const onSubmit = (values: z.infer<typeof domainSchema>) => {
        router.post('/domains', values)
    }
    
    return (
        <Form {...form}>
            <form onSubmit={form.handleSubmit(onSubmit)}>
                <FormField
                    control={form.control}
                    name="url"
                    render={({ field }) => (
                        <FormItem>
                            <FormLabel>Domain URL</FormLabel>
                            <FormControl>
                                <Input placeholder="https://example.com" {...field} />
                            </FormControl>
                            <FormMessage />
                        </FormItem>
                    )}
                />
                <Button type="submit" disabled={form.formState.isSubmitting}>
                    Add Domain
                </Button>
            </form>
        </Form>
    )
}
```

## Error Handling Patterns

### 1. Graceful API Error Handling
```php
// In Controllers
class DomainScanController extends Controller {
    public function store(Domain $domain) {
        try {
            $scan = $domain->scans()->create(['status' => 'pending']);
            RunDomainScan::dispatch($scan);
            
            return response()->json([
                'message' => 'Scan started successfully',
                'scan_id' => $scan->id
            ]);
        } catch (Exception $e) {
            Log::error('Scan initiation failed', [
                'domain_id' => $domain->id,
                'error' => $e->getMessage()
            ]);
            
            return response()->json([
                'message' => 'Failed to start scan',
                'error' => app()->environment('production') ? 'Internal server error' : $e->getMessage()
            ], 500);
        }
    }
}
```

### 2. Scanner Error Handling
```php
class SslTlsScanner implements Scanner {
    public function scan(string $url, array $context = []): ModuleResult {
        try {
            NetworkGuard::assertPublicHttpUrl($url);
            $result = $this->performSslAnalysis($url);
            
            return new ModuleResult(
                module: 'ssl_tls',
                score: $result->score,
                status: 'ok',
                raw: $result->data
            );
        } catch (NetworkGuardException $e) {
            Log::warning('SSRF attempt blocked', ['url' => $url, 'error' => $e->getMessage()]);
            
            return new ModuleResult(
                module: 'ssl_tls',
                score: 0,
                status: 'error',
                raw: ['error' => 'Invalid or unsafe URL']
            );
        } catch (Exception $e) {
            Log::error('SSL scan failed', ['url' => $url, 'error' => $e->getMessage()]);
            
            return new ModuleResult(
                module: 'ssl_tls',
                score: 0,
                status: 'error',
                raw: ['error' => 'Scan failed']
            );
        }
    }
}
```

### 3. Frontend Error Handling
```typescript
// Global error handler for Inertia
import { router } from '@inertiajs/react'
import { toast } from 'sonner'

router.on('error', (event) => {
    if (event.detail.response.status === 429) {
        toast.error('Too many requests. Please wait and try again.')
    } else if (event.detail.response.status >= 500) {
        toast.error('Server error. Please try again later.')
    }
})

// Component error boundaries
export function ScanResults({ domain }: { domain: Domain }) {
    const [error, setError] = useState<string | null>(null)
    
    const triggerScan = async () => {
        try {
            setError(null)
            await router.post(`/domains/${domain.id}/scan`)
            toast.success('Scan started successfully')
        } catch (err) {
            setError('Failed to start scan')
            toast.error('Failed to start scan')
        }
    }
    
    if (error) {
        return (
            <Alert variant="destructive">
                <AlertDescription>{error}</AlertDescription>
            </Alert>
        )
    }
    
    // ... rest of component
}
```

## Testing Best Practices

### 1. Test Structure (AAA Pattern)
```php
class SslTlsScannerTest extends TestCase {
    /** @test */
    public function it_detects_expired_certificates() {
        // Arrange
        $scanner = app(SslTlsScanner::class);
        $expiredUrl = 'https://expired.badssl.com';
        
        // Act
        $result = $scanner->scan($expiredUrl);
        
        // Assert
        $this->assertEquals(0, $result->score);
        $this->assertEquals('fail', $result->status);
        $this->assertArrayHasKey('error', $result->raw);
    }
    
    /** @test */
    public function it_blocks_private_ip_addresses() {
        // Arrange
        $scanner = app(SslTlsScanner::class);
        $privateUrl = 'https://192.168.1.1';
        
        // Act & Assert
        $this->expectException(NetworkGuardException::class);
        $scanner->scan($privateUrl);
    }
}
```

### 2. Feature Testing with Authentication
```php
class DomainManagementTest extends TestCase {
    use RefreshDatabase;
    
    /** @test */
    public function user_can_add_domain_within_limit() {
        // Arrange
        $user = User::factory()->create();
        $this->actingAs($user);
        
        // Act
        $response = $this->post('/domains', [
            'url' => 'https://example.com',
            'email_mode' => 'expected'
        ]);
        
        // Assert
        $response->assertRedirect('/domains');
        $this->assertDatabaseHas('domains', [
            'user_id' => $user->id,
            'url' => 'example.com' // Normalized
        ]);
    }
    
    /** @test */
    public function user_cannot_add_domain_over_limit() {
        // Arrange
        $user = User::factory()->create();
        $user->domains()->createMany(
            factory(Domain::class, 10)->make()->toArray()
        );
        $this->actingAs($user);
        
        // Act
        $response = $this->post('/domains', [
            'url' => 'https://newdomain.com'
        ]);
        
        // Assert
        $response->assertSessionHasErrors('url');
        $this->assertDatabaseMissing('domains', ['url' => 'newdomain.com']);
    }
}
```

### 3. E2E Testing with Playwright MCP
```typescript
test('user can complete full scan workflow', async ({ page }) => {
    // Arrange - Create user and login
    await page.goto('/login')
    await page.fill('[name="email"]', 'test@example.com')
    await page.fill('[name="password"]', 'password')
    await page.getByRole('button', { name: 'Login' }).click()
    
    // Act - Add domain and trigger scan
    await page.goto('/domains')
    await page.getByRole('button', { name: 'Add Domain' }).click()
    
    await page.fill('[name="url"]', 'https://example.com')
    await page.getByRole('button', { name: 'Add & Scan' }).click()
    
    // Assert - Verify scan completion
    await page.waitForSelector('[data-scan-status="completed"]', { timeout: 30000 })
    
    const scoreElement = await page.getByTestId('security-score')
    await expect(scoreElement).toBeVisible()
    
    const score = await scoreElement.textContent()
    expect(parseInt(score!)).toBeGreaterThanOrEqual(0)
    expect(parseInt(score!)).toBeLessThanOrEqual(100)
})
```

## Command Reference

### Development Commands
```bash
# Start development
php artisan serve
npm run dev

# Run tests (always run before commits)
php artisan pest --parallel
npm run type-check
npx playwright test

# Code quality
php artisan pint
npm run lint

# Database
php artisan migrate:fresh --seed
php artisan tinker
```

### Security Testing Commands
```bash
# Test with various SSL configurations
curl -I https://badssl.com
curl -I https://expired.badssl.com
curl -I https://wrong.host.badssl.com
curl -I https://self-signed.badssl.com
curl -I https://untrusted-root.badssl.com

# Test security headers
curl -I https://example.com
curl -I https://github.com  # Good headers example
curl -I https://neverssl.com  # Minimal headers example

# Test DNS configurations
dig example.com A
dig example.com MX
dig _dmarc.example.com TXT
dig default._domainkey.example.com TXT
dig example.com CAA

# Test rate limiting (should hit limit after configured threshold)
for i in {1..15}; do curl -H "Authorization: Bearer $token" http://localhost:8000/api/domains/1/scan; done

# Test input validation and SSRF protection
curl -X POST -H "Content-Type: application/json" \
  -d '{"url":"http://192.168.1.1"}' \
  http://localhost:8000/api/domains

curl -X POST -H "Content-Type: application/json" \
  -d '{"url":"http://127.0.0.1"}' \
  http://localhost:8000/api/domains

curl -X POST -H "Content-Type: application/json" \
  -d '{"url":"http://169.254.169.254"}' \
  http://localhost:8000/api/domains

# Test timeout handling (use a slow/unresponsive service)
curl -X POST -H "Content-Type: application/json" \
  -d '{"url":"https://httpbin.org/delay/60"}' \
  http://localhost:8000/api/domains
```

### Manual Verification Commands
```bash
# Verify domain creation
php artisan tinker
>>> User::find(1)->domains()->count()
>>> Domain::where('url', 'example.com')->first()

# Verify scan execution
>>> $domain = Domain::first()
>>> $scan = $domain->scans()->create(['status' => 'pending'])
>>> \App\Jobs\RunDomainScan::dispatch($scan)

# Test individual scanners
>>> $sslScanner = app(\App\Services\Scanners\SslTlsScanner::class)
>>> $result = $sslScanner->scan('https://github.com')
>>> $result->score  // Should show SSL score
>>> $result->raw    // Should show detailed SSL analysis

>>> $headersScanner = app(\App\Services\Scanners\SecurityHeadersScanner::class)  
>>> $result = $headersScanner->scan('https://github.com')
>>> $result->raw['headers_found']  // Should show security headers

>>> $dnsScanner = app(\App\Services\Scanners\DnsEmailScanner::class)
>>> $result = $dnsScanner->scan('https://github.com', ['email_mode' => 'expected'])
>>> $result->raw['email_security']  // Should show email security analysis

# Verify rate limiting
>>> app('cache')->get('throttle:scans:1')
>>> app('cache')->get('scanner_rate_limit:ssl_tls:github.com')

# Verify error handling
>>> $sslScanner->scan('https://192.168.1.1')  // Should throw SSRFException
>>> $sslScanner->scan('https://expired.badssl.com')  // Should return low score

# Check scan results in database
>>> $scan = \App\Models\Scan::with('modules')->latest()->first()
>>> $scan->modules->pluck('score', 'module')  // Should show scores per module
>>> $scan->total_score  // Should show weighted total
>>> $scan->grade  // Should show letter grade
```

Remember: **Security first, test everything, trust nothing from external sources.**

## Additional SaaS Systems Implementation

### Email Communication System
- **Welcome Sequence**: Automated emails for user activation (registration → first domain → first scan)
- **Trial Management**: Smart reminders at days 7, 10, 12, 14 with user progress context
- **Scan Notifications**: Completion alerts with score summary and critical issue warnings
- **Billing Communications**: Payment confirmations, failures, and subscription changes
- **Implementation**: Laravel Mail + Queue jobs with preference management

### Support Infrastructure
- **Contact Form**: Multi-category routing (billing/technical/feature/other) with priority levels
- **FAQ System**: Searchable knowledge base with voting and analytics
- **Feedback Widget**: Contextual in-app feedback collection with screenshot capture
- **Ticket Management**: Support ticket system with auto-responses and routing
- **Implementation**: Forms + Knowledge base + Support ticket models

### Legal & Compliance Pages
- **Terms of Service**: Domain ownership requirements, acceptable use, liability limitations
- **Privacy Policy**: GDPR-compliant data handling, user rights, transparent practices
- **Cookie Consent**: Preference management with essential/analytics/marketing categories
- **Legal Framework**: Document versioning, acceptance tracking, compliance automation
- **Implementation**: Legal document models + consent tracking + GDPR user controls

### Guided Onboarding Flow
- **Welcome Experience**: Interactive intro with personalization and value demonstration
- **Progressive Disclosure**: Feature introduction based on user behavior and milestones
- **Achievement System**: Gamification with points, badges, and milestone celebrations
- **Analytics Integration**: Conversion funnel tracking and A/B testing framework
- **Implementation**: React components + achievement tracking + onboarding analytics

### Integration Points
- Add to **Phase 9** of execution plan (Day 17-18)
- Database additions: notification_preferences, support_tickets, legal_documents, user_achievements
- UI components using Shadcn/ui for consistency
- Queue jobs for email processing and notifications
- Analytics tracking for optimization and conversion measurement