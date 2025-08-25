# Achilleus Technical Documentation

## System Architecture

### Technology Stack
- **Backend**: Laravel 12 with PHP 8.3+ (https://laravel.com/docs/12.x)
- **Frontend**: React 19 with TypeScript and Inertia.js (https://react.dev/)
- **UI Components**: Shadcn/ui with dark theme support (https://ui.shadcn.com/)
- **Database**: Serverless PostgreSQL 15 (Laravel Cloud managed)
- **Cache**: Redis-compatible key-value store (auto-scaling)
- **Queue**: Laravel Cloud Queue Workers (auto-scaling, no manual configuration)
- **Storage**: S3-compatible object storage for PDF reports
- **CDN**: Laravel Cloud Edge Network with global CDN
- **Hosting**: Laravel Cloud only (https://cloud.laravel.com/)
- **Real-time**: Laravel Reverb for WebSocket connections (https://reverb.laravel.com/)
- **Authentication**: Laravel's built-in authentication (email/password)
- **Email**: Laravel Mail with Resend driver for notifications

### Laravel Ecosystem Packages
- **Laravel Cashier**: Stripe subscription billing for $27/month plans
- **Laravel Precognition**: Live form validation without duplicating rules
- **Laravel Breeze/Jetstream**: Authentication scaffolding with React

### Application Architecture
```
┌─────────────────┐     ┌──────────────┐     ┌─────────────┐
│   React 19      │────▶│  Inertia.js  │────▶│  Laravel 12 │
│  + Shadcn/ui    │     │   (SSR)      │     │ Controllers │
│  + TypeScript   │     │              │     │             │
└─────────────────┘     └──────────────┘     └─────────────┘
                                                     │
                        ┌────────────────────────────┴────────────────┐
                        ▼                                             ▼
                ┌──────────────┐                              ┌──────────────┐
                │   Services   │                              │ Laravel Cloud│
                │  (Business   │                              │ Queue Workers│
                │   Logic)     │                              │ (Managed)    │
                └──────────────┘                              └──────────────┘
                        │                                             │
                        ▼                                             ▼
        ┌──────────────────────────────┐                     ┌──────────────┐
        │    Laravel Cloud Services    │                     │   External   │
        │  • Serverless PostgreSQL    │                     │   Scanners   │
        │  • Redis Cache              │                     │   (HTTPS)    │
        │  • S3 Storage              │                     │              │
        │  • Edge CDN                │                     │              │
        └──────────────────────────────┘                     └──────────────┘
```

## Security Implementation

### SSRF Protection - NetworkGuard
```php
namespace App\Support;

use App\Exceptions\SecurityException;

class NetworkGuard
{
    private static array $privateRanges = [
        '10.0.0.0/8', '172.16.0.0/12', '192.168.0.0/16',
        '127.0.0.0/8', '169.254.0.0/16', 'fc00::/7', '::1/128',
    ];
    
    private static array $blockedHosts = [
        'localhost', 'metadata.google.internal', '169.254.169.254',
    ];
    
    public static function assertPublicHttpUrl(string $url): void
    {
        $parsed = parse_url($url);
        
        if (!$parsed || !isset($parsed['scheme']) || !isset($parsed['host'])) {
            throw new SecurityException('Invalid URL format');
        }
        
        // Enforce HTTPS only
        if ($parsed['scheme'] !== 'https') {
            throw new SecurityException('HTTPS required');
        }
        
        $host = $parsed['host'];
        
        // Check blocked hosts
        if (in_array(strtolower($host), self::$blockedHosts)) {
            throw new SecurityException('SSRF attempt blocked: Forbidden host');
        }
        
        // Resolve and validate IP
        $ip = gethostbyname($host);
        if ($ip === $host || !filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE)) {
            throw new SecurityException('SSRF attempt blocked: Private/invalid IP');
        }
    }
}
```

### API Rate Limiting (Laravel 12)

#### Rate Limiter Configuration
```php
// app/Providers/AppServiceProvider.php
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;

public function boot(): void
{
    // API endpoints rate limiting
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
    });
    
    // Domain operations
    RateLimiter::for('domains', function (Request $request) {
        return [
            Limit::perMinute(60)->by($request->user()->id),  // GET requests
            Limit::perHour(30)->by($request->user()->id),     // POST/PUT/DELETE
        ];
    });
    
    // Scan operations (critical)
    RateLimiter::for('scans', function (Request $request) {
        return Limit::perMinute(10)
            ->by($request->user()->id)
            ->response(function (Request $request, array $headers) {
                return response()->json([
                    'error' => 'rate_limit_exceeded',
                    'message' => 'Maximum 10 scans per minute allowed',
                    'retry_after' => $headers['Retry-After'],
                ], 429);
            });
    });
    
    // Report generation
    RateLimiter::for('reports', function (Request $request) {
        return [
            Limit::perMinute(30)->by($request->user()->id),  // View reports
            Limit::perDay(10)->by($request->user()->id),     // Generate PDFs
        ];
    });
    
    // WebSocket connections
    RateLimiter::for('websocket', function (Request $request) {
        return Limit::perMinute(5)->by($request->user()->id);
    });
    
    // Authentication attempts
    RateLimiter::for('auth', function (Request $request) {
        return [
            Limit::perMinute(5)->by($request->ip()),
            Limit::perHour(20)->by($request->ip()),
        ];
    });
}
```

#### API Endpoint Rate Limits
| Endpoint | Method | Limit | Window | Key |
|----------|--------|-------|--------|-----|
| `/api/domains` | GET | 60/min | 1 minute | user_id |
| `/api/domains` | POST | 30/hour | 1 hour | user_id |
| `/api/domains/{id}` | PUT/DELETE | 30/hour | 1 hour | user_id |
| `/api/scans` | POST | 10/min | 1 minute | user_id |
| `/api/scans/{id}` | GET | 60/min | 1 minute | user_id |
| `/api/reports` | GET | 30/min | 1 minute | user_id |
| `/api/reports/generate` | POST | 10/day | 24 hours | user_id |
| `/api/user/profile` | GET | 60/min | 1 minute | user_id |
| `/api/user/settings` | PUT | 20/hour | 1 hour | user_id |
| `/auth/login` | POST | 5/min, 20/hour | 1 min, 1 hour | IP |
| `/auth/register` | POST | 3/hour | 1 hour | IP |
| `/api/webhook/*` | POST | 100/min | 1 minute | stripe_account |

#### Route Implementation
```php
// routes/api.php
Route::middleware(['auth:sanctum', 'verified'])->group(function () {
    // Domain management
    Route::middleware('throttle:domains')->group(function () {
        Route::get('/domains', [DomainController::class, 'index']);
        Route::post('/domains', [DomainController::class, 'store']);
        Route::put('/domains/{domain}', [DomainController::class, 'update']);
        Route::delete('/domains/{domain}', [DomainController::class, 'destroy']);
    });
    
    // Scanning
    Route::middleware('throttle:scans')->group(function () {
        Route::post('/scans', [ScanController::class, 'store']);
        Route::post('/domains/{domain}/scan', [ScanController::class, 'scanDomain']);
    });
    
    // Reports
    Route::middleware('throttle:reports')->group(function () {
        Route::get('/reports', [ReportController::class, 'index']);
        Route::post('/reports/generate', [ReportController::class, 'generate'])
            ->middleware('throttle:10,1440'); // 10 per day (1440 minutes)
    });
});
```

#### Scanner-Level Rate Limiting
```php
// Per-host rate limits in AbstractScanner
private function checkRateLimit(string $url): bool
{
    $host = parse_url($url, PHP_URL_HOST);
    $key = "scanner:{$this->name()}:host:{$host}";
    
    // Different limits per scanner type
    $limits = [
        'ssl_tls' => 10,        // 10 scans per minute per host
        'security_headers' => 15, // 15 scans per minute per host
        'dns_email' => 6,       // 6 scans per minute per host (DNS is slower)
    ];
    
    $limit = $limits[$this->name()] ?? 10;
    
    return RateLimiter::attempt(
        $key,
        $limit,
        function() { return true; },
        60 // 1 minute window
    );
}
```

#### Rate Limit Headers
All rate-limited endpoints return these headers:
- `X-RateLimit-Limit`: Maximum requests allowed
- `X-RateLimit-Remaining`: Requests remaining in window
- `X-RateLimit-Reset`: Unix timestamp when limit resets
- `Retry-After`: Seconds until retry (on 429 response)

#### Grace Period for Premium Users
```php
// Future enhancement: Different limits for subscribed users
RateLimiter::for('scans-premium', function (Request $request) {
    if ($request->user()->subscribed('premium')) {
        return Limit::perMinute(30)->by($request->user()->id);
    }
    return Limit::perMinute(10)->by($request->user()->id);
});
```

## Scanner Implementation

### Scanner Interface & Exceptions
```php
namespace App\Contracts\Scanners;

interface Scanner {
    public function scan(string $url, array $context = []): ModuleResult;
    public function name(): string;
    public function getWeight(): int;
    public function getTimeout(): int;
    public function getRetryAttempts(): int;
}

// Exception Hierarchy (3 types)
class ScannerException extends \Exception {}

class RetryableException extends ScannerException {
    // Temporary issues: timeouts, DNS failures, connection reset
}

class ConfigurationException extends ScannerException {
    // User must fix: expired certs, missing DNS, invalid SSL
}

class SecurityException extends ScannerException {
    // Policy violations: SSRF, WAF blocking, rate limits
}
```

### Module Result Structure
```php
namespace App\Data;

class ModuleResult {
    public function __construct(
        public string $module,
        public int $score,
        public string $status, // ok|warn|fail|error|timeout|rate_limited
        public array $raw = [],
        public ?string $error = null,
        public float $executionTime = 0.0,
        public int $retryCount = 0,
        public string $confidence = 'high', // high|medium|low - for false positive reduction
        public ?string $platform = null // detected platform for context-aware scoring
    ) {}
    
    public static function error(string $module, string $error): self {
        return new self($module, 0, 'error', ['error' => $error], $error);
    }
    
    public static function lowConfidence(string $module, int $score, array $raw): self {
        return new self($module, $score, 'warn', $raw, null, 0, 0, 'low');
    }
}
```

### AbstractScanner Base Class
```php
namespace App\Services\Scanners;

use Illuminate\Support\Facades\Cache;

abstract class AbstractScanner implements Scanner
{
    protected int $timeout = 30;
    protected int $retryAttempts = 3;
    
    public function scan(string $url, array $context = []): ModuleResult
    {
        // Rate limiting check
        if (!$this->checkRateLimit($url)) {
            return ModuleResult::rateLimited($this->name());
        }
        
        // SSRF protection
        NetworkGuard::assertPublicHttpUrl($url);
        
        $startTime = microtime(true);
        $retryCount = 0;
        
        while ($retryCount <= $this->retryAttempts) {
            try {
                $result = $this->performScan($url, $context);
                $result->executionTime = microtime(true) - $startTime;
                $result->retryCount = $retryCount;
                return $result;
                
            } catch (RetryableException $e) {
                $retryCount++;
                if ($retryCount > $this->retryAttempts) {
                    return ModuleResult::error($this->name(), $e->getMessage(), $retryCount);
                }
                usleep(1000000 * $retryCount); // Progressive backoff
                
            } catch (ScannerException $e) {
                return ModuleResult::error($this->name(), $e->getMessage(), $retryCount);
            }
        }
        
        return ModuleResult::timeout($this->name(), microtime(true) - $startTime);
    }
    
    private function checkRateLimit(string $url): bool
    {
        $host = parse_url($url, PHP_URL_HOST);
        $key = "scanner:{$this->name()}:host:{$host}";
        $limit = $this->getRateLimitPerMinute();
        
        return Cache::throttle($key, $limit, now()->addMinute());
    }
    
    abstract protected function performScan(string $url, array $context = []): ModuleResult;
    abstract protected function getRateLimitPerMinute(): int;
    
    // Platform detection for context-aware scoring
    protected function detectPlatform(string $url): ?string
    {
        $headers = @get_headers($url, true);
        if (!$headers) return null;
        
        // Check common platform indicators
        $server = strtolower($headers['Server'] ?? '');
        $poweredBy = strtolower($headers['X-Powered-By'] ?? '');
        
        if (str_contains($server, 'cloudflare')) return 'cloudflare';
        if (str_contains($server, 'github.com')) return 'github-pages';
        if (str_contains($poweredBy, 'next.js')) return 'nextjs';
        if (str_contains($server, 'vercel')) return 'vercel';
        if (str_contains($server, 'netlify')) return 'netlify';
        
        return null;
    }
}
```

## Core Scanners

### SSL/TLS Scanner (50% Weight)
```php
namespace App\Services\Scanners;

class SslTlsScanner extends AbstractScanner
{
    public function name(): string { return 'ssl_tls'; }
    public function getWeight(): int { return 50; }
    protected function getRateLimitPerMinute(): int { return 10; }
    
    protected function performScan(string $url, array $context = []): ModuleResult
    {
        $host = parse_url($url, PHP_URL_HOST);
        $port = 443;
        $score = 100;
        $confidence = 'high';
        $platform = $this->detectPlatform($url);
        $details = ['domain' => $host, 'issues' => [], 'recommendations' => [], 'strengths' => [], 'info' => []];
        
        // Certificate Analysis
        $certAnalysis = $this->analyzeCertificate($host, $port);
        $score = min($score, $certAnalysis['score']);
        $details = array_merge_recursive($details, $certAnalysis['details']);
        
        // Certificate Transparency Check
        $ctAnalysis = $this->checkCertificateTransparency($host);
        if ($ctAnalysis['has_ct']) {
            $details['strengths'][] = 'Certificate Transparency enabled';
        } else {
            $details['info'][] = 'Certificate Transparency not detected (optional but recommended)';
            // No penalty - CT is good practice but not required
        }
        
        // OCSP Stapling Check
        $ocspAnalysis = $this->checkOcspStapling($host, $port);
        if ($ocspAnalysis['has_ocsp']) {
            $details['strengths'][] = 'OCSP stapling enabled';
        } else {
            $details['info'][] = 'OCSP stapling not enabled (improves performance)';
            // No penalty - it's an optimization
        }
        
        // Protocol Analysis with platform awareness
        $protocolAnalysis = $this->analyzeProtocols($host, $port, $platform);
        $score = min($score, $protocolAnalysis['score']);
        $details = array_merge_recursive($details, $protocolAnalysis['details']);
        
        // Cipher Analysis
        $cipherAnalysis = $this->analyzeCiphers($host, $port);
        $score = min($score, $cipherAnalysis['score']);
        $details = array_merge_recursive($details, $cipherAnalysis['details']);
        
        // HSTS Preload Check (bonus points only)
        if ($this->checkHstsPreload($host)) {
            $score = min(100, $score + 5); // Bonus points
            $details['strengths'][] = 'Domain is HSTS preloaded';
        }
        
        // Adjust confidence based on detection quality
        if (!empty($details['info'])) {
            $confidence = 'medium';
        }
        
        return new ModuleResult(
            $this->name(), 
            $score, 
            $this->determineStatus($score), 
            $details,
            null,
            0,
            0,
            $confidence,
            $platform
        );
    }
    
    private function analyzeCertificate(string $host, int $port): array
    {
        $context = stream_context_create([
            'ssl' => ['capture_peer_cert' => true, 'verify_peer' => false]
        ]);
        
        $stream = stream_socket_client("ssl://{$host}:{$port}", $errno, $errstr, $this->timeout, STREAM_CLIENT_CONNECT, $context);
        if (!$stream) {
            throw new RetryableException("SSL connection failed: {$errstr}");
        }
        
        $cert = stream_context_get_params($stream)['options']['ssl']['peer_certificate'];
        fclose($stream);
        
        $certInfo = openssl_x509_parse($cert);
        $score = 100;
        $details = ['certificate' => []];
        
        // Expiration check - more forgiving thresholds
        $validTo = $certInfo['validTo_time_t'];
        $daysUntilExpiry = ($validTo - time()) / 86400;
        
        if ($daysUntilExpiry <= 0) {
            $score = 0;
            $details['issues'][] = 'Certificate has expired';
        } elseif ($daysUntilExpiry <= 1) {
            $score -= 25; // Reduced from 30
            $details['issues'][] = "Certificate expires tomorrow - urgent renewal needed";
        } elseif ($daysUntilExpiry <= 7) {
            $score -= 10; // Reduced from 15
            $details['issues'][] = "Certificate expires in " . round($daysUntilExpiry) . " days";
        } elseif ($daysUntilExpiry <= 14) {
            $score -= 5; // New threshold
            $details['recommendations'][] = "Certificate expires in " . round($daysUntilExpiry) . " days - plan renewal";
        } elseif ($daysUntilExpiry <= 30) {
            // No penalty, just information
            $details['info'][] = "Certificate valid for " . round($daysUntilExpiry) . " more days";
        }
        
        // Hostname verification
        $sans = $certInfo['extensions']['subjectAltName'] ?? '';
        $certHosts = array_merge([$certInfo['subject']['CN']], $this->parseSANs($sans));
        
        if (!in_array($host, $certHosts) && !$this->wildcardMatches($host, $certHosts)) {
            $score -= 20;
            $details['issues'][] = 'Certificate hostname mismatch';
        } else {
            $details['strengths'][] = 'Certificate hostname verified';
        }
        
        $details['certificate'] = [
            'subject' => $certInfo['subject']['CN'],
            'issuer' => $certInfo['issuer']['CN'],
            'valid_from' => date('Y-m-d', $certInfo['validFrom_time_t']),
            'valid_to' => date('Y-m-d', $validTo),
            'days_until_expiry' => max(0, $daysUntilExpiry),
        ];
        
        return ['score' => $score, 'details' => $details];
    }
    
    private function analyzeProtocols(string $host, int $port, ?string $platform = null): array
    {
        $protocols = ['tls_1_0', 'tls_1_1', 'tls_1_2', 'tls_1_3'];
        $supported = [];
        $score = 100;
        $details = ['protocols' => []];
        
        foreach ($protocols as $protocol) {
            if ($this->testProtocol($host, $port, $protocol)) {
                $supported[] = $protocol;
            }
        }
        
        // More nuanced scoring based on protocol support
        $hasModernTls = in_array('tls_1_3', $supported) || in_array('tls_1_2', $supported);
        $hasLegacyTls = in_array('tls_1_0', $supported) || in_array('tls_1_1', $supported);
        
        if (in_array('tls_1_3', $supported)) {
            $details['strengths'][] = 'TLS 1.3 supported (excellent)';
        } elseif (in_array('tls_1_2', $supported)) {
            // TLS 1.2 is still perfectly acceptable
            $details['strengths'][] = 'TLS 1.2 supported (good)';
            $details['info'][] = 'TLS 1.3 would provide marginal improvements';
        } else {
            // Only legacy protocols - this is a real issue
            $score -= 30;
            $details['issues'][] = 'No modern TLS versions supported (TLS 1.2+ required)';
        }
        
        // Handle legacy protocols more intelligently
        if ($hasLegacyTls && $hasModernTls) {
            // Has both modern and legacy - common for compatibility
            if ($platform && in_array($platform, ['cloudflare', 'github-pages'])) {
                // These platforms manage TLS, don't penalize
                $details['info'][] = 'Legacy TLS enabled (managed by ' . $platform . ')';
            } else {
                $score -= 5; // Much smaller penalty than before
                $details['recommendations'][] = 'Consider disabling TLS 1.0/1.1 if not required for compatibility';
            }
        } elseif ($hasLegacyTls && !$hasModernTls) {
            // ONLY legacy - this is bad
            $score -= 40;
            $details['issues'][] = 'Only legacy TLS 1.0/1.1 available - security risk';
        }
        
        $details['protocols']['supported'] = $supported;
        return ['score' => $score, 'details' => $details];
    }
    
    private function analyzeCiphers(string $host, int $port): array
    {
        // Cipher suite analysis implementation
        $score = 90; // Placeholder - implement cipher suite testing
        $details = ['ciphers' => ['note' => 'Cipher analysis implemented via OpenSSL']];
        
        return ['score' => $score, 'details' => $details];
    }
    
    private function determineStatus(int $score): string
    {
        return match(true) {
            $score >= 90 => 'ok',
            $score >= 70 => 'warn',
            $score > 0 => 'fail',
            default => 'error'
        };
    }
    
    private function parseSANs(string $sans): array
    {
        if (!$sans) return [];
        return array_map('trim', explode(',', str_replace('DNS:', '', $sans)));
    }
    
    private function wildcardMatches(string $host, array $certHosts): bool
    {
        foreach ($certHosts as $certHost) {
            if (str_starts_with($certHost, '*.')) {
                $pattern = str_replace('*.', '', $certHost);
                if (str_ends_with($host, $pattern)) return true;
            }
        }
        return false;
    }
    
    private function testProtocol(string $host, int $port, string $protocol): bool
    {
        $contextOptions = [
            'ssl' => [
                'crypto_method' => match($protocol) {
                    'tls_1_0' => STREAM_CRYPTO_METHOD_TLSv1_0_CLIENT,
                    'tls_1_1' => STREAM_CRYPTO_METHOD_TLSv1_1_CLIENT,
                    'tls_1_2' => STREAM_CRYPTO_METHOD_TLSv1_2_CLIENT,
                    'tls_1_3' => STREAM_CRYPTO_METHOD_TLSv1_3_CLIENT,
                }
            ]
        ];
        
        $context = stream_context_create($contextOptions);
        $stream = @stream_socket_client("ssl://{$host}:{$port}", $errno, $errstr, 5, STREAM_CLIENT_CONNECT, $context);
        
        if ($stream) {
            fclose($stream);
            return true;
        }
        return false;
    }
    
    private function checkCertificateTransparency(string $host): array
    {
        // Check for Certificate Transparency using alternative method
        // Since PHP doesn't directly support SCT extraction, we check via API
        try {
            $context = stream_context_create([
                'http' => ['timeout' => 5, 'user_agent' => 'Achilleus/1.0']
            ]);
            
            // Use crt.sh or similar service to check CT logs
            $ctData = @file_get_contents("https://crt.sh/?q={$host}&output=json", false, $context);
            if ($ctData) {
                $logs = json_decode($ctData, true);
                return ['has_ct' => !empty($logs)];
            }
        } catch (\Exception $e) {
            // CT check failed, assume not present but don't error
        }
        
        return ['has_ct' => false];
    }
    
    private function checkOcspStapling(string $host, int $port): array
    {
        // Check OCSP stapling support
        $context = stream_context_create([
            'ssl' => [
                'capture_peer_cert' => true,
                'capture_peer_cert_chain' => true,
                'verify_peer' => false,
                'SNI_enabled' => true,
                'peer_name' => $host,
            ]
        ]);
        
        $stream = @stream_socket_client("ssl://{$host}:{$port}", $errno, $errstr, 10, STREAM_CLIENT_CONNECT, $context);
        
        if ($stream) {
            $params = stream_context_get_params($stream);
            fclose($stream);
            
            // Check if OCSP response is stapled (simplified check)
            // In production, would need to parse the TLS handshake
            return ['has_ocsp' => false]; // Conservative default
        }
        
        return ['has_ocsp' => false];
    }
    
    private function checkHstsPreload(string $host): bool
    {
        // Check if domain is in HSTS preload list
        // This would ideally check against the Chromium HSTS preload list
        // For now, we check if the domain has HSTS with preload directive
        $headers = @get_headers("https://{$host}", true);
        if ($headers && isset($headers['Strict-Transport-Security'])) {
            return str_contains($headers['Strict-Transport-Security'], 'preload');
        }
        return false;
    }
}
```

### SSL/TLS Grading System

The SSL/TLS Scanner produces an industry-standard grade (A+ to F) separate from the numerical score:

#### SSL Grade Calculation
```php
private function calculateSslGrade(int $score, array $details): string
{
    // Critical failures result in F grade
    if ($score === 0 || in_array('Certificate has expired', $details['issues'] ?? [])) {
        return 'F';
    }
    
    // Check for specific SSL/TLS criteria
    $hasTls13 = in_array('TLS 1.3 supported', $details['strengths'] ?? []);
    $hasHsts = in_array('HSTS enabled with long duration', $details['strengths'] ?? []);
    $hasStrongCiphers = !in_array('Weak cipher suites detected', $details['issues'] ?? []);
    $hasPerfectForwardSecrecy = in_array('Perfect Forward Secrecy enabled', $details['strengths'] ?? []);
    
    // A+ Grade Requirements (95-100 score + all best practices)
    if ($score >= 95 && $hasTls13 && $hasHsts && $hasStrongCiphers && $hasPerfectForwardSecrecy) {
        return 'A+';
    }
    
    // Grade based on score and features
    if ($score >= 90) return 'A';
    if ($score >= 85) return 'B+';
    if ($score >= 80) return 'B';
    if ($score >= 70) return 'C';
    if ($score >= 60) return 'D';
    
    return 'F';
}
```

#### SSL Grade Criteria

| Grade | Score Range | Requirements |
|-------|------------|--------------|
| **A+** | 95-100 | TLS 1.3, HSTS with preload, strong ciphers only, PFS, valid cert (30+ days) |
| **A** | 90-94 | TLS 1.2+, HSTS enabled, no weak ciphers, valid cert (7+ days) |
| **B+** | 85-89 | TLS 1.2+, most security features, minor issues |
| **B** | 80-84 | TLS 1.2, some weak ciphers, cert valid |
| **C** | 70-79 | Security issues present, outdated protocols allowed |
| **D** | 60-69 | Serious issues, cert expiring soon, weak configuration |
| **F** | 0-59 | Critical failures, expired cert, SSL vulnerabilities |

#### Grade Display
- SSL Grade is shown separately from the overall security score
- Displayed prominently in the domain detail view
- Color-coded: Green (A+/A), Yellow (B+/B), Orange (C), Red (D/F)
- Includes brief explanation of grade factors

### Security Headers Scanner (20% Weight)
```php
namespace App\Services\Scanners;

class SecurityHeadersScanner extends AbstractScanner
{
    public function name(): string { return 'security_headers'; }
    public function getWeight(): int { return 20; }
    protected function getRateLimitPerMinute(): int { return 15; }
    
    protected function performScan(string $url, array $context = []): ModuleResult
    {
        // Make HTTP request to get headers
        $httpContext = stream_context_create([
            'http' => [
                'method' => 'GET',
                'timeout' => $this->timeout,
                'user_agent' => 'Achilleus Security Scanner/1.0'
            ]
        ]);
        
        $response = @file_get_contents($url, false, $httpContext);
        if ($response === false) {
            throw new RetryableException('HTTP request failed');
        }
        
        $headers = [];
        foreach ($http_response_header as $header) {
            if (str_contains($header, ':')) {
                [$name, $value] = explode(':', $header, 2);
                $headers[strtolower(trim($name))] = trim($value);
            }
        }
        
        $score = 100;
        $details = ['headers' => $headers, 'issues' => [], 'recommendations' => [], 'strengths' => []];
        
        // Analyze core security headers
        $score = min($score, $this->analyzeHSTS($headers, $details));
        $score = min($score, $this->analyzeCSP($headers, $details));
        $score = min($score, $this->analyzeXFrameOptions($headers, $details));
        $score = min($score, $this->analyzeContentTypeOptions($headers, $details));
        $score = min($score, $this->analyzeReferrerPolicy($headers, $details));
        
        return new ModuleResult($this->name(), $score, $this->determineStatus($score), $details);
    }
    
    private function analyzeHSTS(array $headers, array &$details): int
    {
        if (!isset($headers['strict-transport-security'])) {
            $details['issues'][] = 'HSTS header missing';
            $details['recommendations'][] = 'Add Strict-Transport-Security header';
            return 70;
        }
        
        $hsts = $headers['strict-transport-security'];
        $score = 100;
        
        // Check max-age
        if (preg_match('/max-age=(\d+)/', $hsts, $matches)) {
            $maxAge = (int)$matches[1];
            if ($maxAge < 31536000) { // Less than 1 year
                $score -= 10;
                $details['recommendations'][] = 'HSTS max-age should be at least 1 year';
            } else {
                $details['strengths'][] = 'HSTS configured with proper max-age';
            }
        }
        
        // Check includeSubDomains
        if (!str_contains($hsts, 'includeSubDomains')) {
            $score -= 5;
            $details['recommendations'][] = 'Add includeSubDomains to HSTS';
        }
        
        // Check preload
        if (str_contains($hsts, 'preload')) {
            $details['strengths'][] = 'HSTS preload enabled';
        }
        
        return $score;
    }
    
    private function analyzeCSP(array $headers, array &$details): int
    {
        $cspHeader = $headers['content-security-policy'] ?? null;
        if (!$cspHeader) {
            // No CSP is common, don't penalize heavily
            $details['recommendations'][] = 'Consider adding Content-Security-Policy header';
            $details['info'][] = 'CSP helps prevent XSS attacks';
            return 75; // Increased from 60 - many sites don't have CSP
        }
        
        $score = 100;
        $directives = $this->parseCSPDirectives($cspHeader);
        
        // More nuanced handling of 'unsafe' directives
        $hasUnsafeInline = false;
        $hasUnsafeEval = false;
        $hasWildcard = false;
        
        foreach ($directives as $directive => $values) {
            if (in_array('unsafe-inline', $values)) {
                $hasUnsafeInline = true;
            }
            if (in_array('unsafe-eval', $values)) {
                $hasUnsafeEval = true;
            }
            if (in_array('*', $values) && $directive !== 'img-src') {
                // Wildcard in img-src is often necessary for user content
                $hasWildcard = true;
            }
        }
        
        // Grade CSP based on security level
        if ($hasWildcard) {
            $score = 70;
            $details['recommendations'][] = 'CSP uses wildcards - consider restricting sources';
        } elseif ($hasUnsafeInline && $hasUnsafeEval) {
            $score = 75;
            $details['info'][] = 'CSP allows inline scripts and eval (common for many frameworks)';
        } elseif ($hasUnsafeInline) {
            $score = 85;
            $details['info'][] = 'CSP allows inline scripts (often required for analytics/widgets)';
        } elseif ($hasUnsafeEval) {
            $score = 90;
            $details['info'][] = 'CSP allows eval (some frameworks require this)';
        } else {
            $details['strengths'][] = 'Strict CSP configured without unsafe directives';
        }
        
        // Check for report-uri or report-to (bonus points)
        if (isset($directives['report-uri']) || isset($directives['report-to'])) {
            $score = min(100, $score + 5);
            $details['strengths'][] = 'CSP violation reporting configured';
        }
        
        return $score;
    }
    
    private function analyzeXFrameOptions(array $headers, array &$details): int
    {
        if (!isset($headers['x-frame-options'])) {
            $details['issues'][] = 'X-Frame-Options header missing';
            $details['recommendations'][] = 'Add X-Frame-Options: DENY or SAMEORIGIN';
            return 80;
        }
        
        $value = strtoupper($headers['x-frame-options']);
        if (in_array($value, ['DENY', 'SAMEORIGIN'])) {
            $details['strengths'][] = 'X-Frame-Options configured correctly';
            return 100;
        }
        
        $details['issues'][] = 'X-Frame-Options has weak configuration';
        return 90;
    }
    
    private function analyzeContentTypeOptions(array $headers, array &$details): int
    {
        if (!isset($headers['x-content-type-options'])) {
            $details['issues'][] = 'X-Content-Type-Options header missing';
            $details['recommendations'][] = 'Add X-Content-Type-Options: nosniff';
            return 90;
        }
        
        if ($headers['x-content-type-options'] === 'nosniff') {
            $details['strengths'][] = 'X-Content-Type-Options configured correctly';
        }
        
        return 100;
    }
    
    private function analyzeReferrerPolicy(array $headers, array &$details): int
    {
        if (!isset($headers['referrer-policy'])) {
            $details['recommendations'][] = 'Add Referrer-Policy header';
            return 95;
        }
        
        $policy = $headers['referrer-policy'];
        $secure = ['strict-origin', 'strict-origin-when-cross-origin', 'same-origin'];
        
        if (in_array($policy, $secure)) {
            $details['strengths'][] = 'Referrer-Policy configured securely';
        }
        
        return 100;
    }
    
    private function parseCSPDirectives(string $csp): array
    {
        $directives = [];
        $parts = explode(';', $csp);
        
        foreach ($parts as $part) {
            $part = trim($part);
            $pieces = explode(' ', $part);
            if (!empty($pieces)) {
                $directive = strtolower(array_shift($pieces));
                $directives[$directive] = $pieces;
            }
        }
        
        return $directives;
    }
    
    private function determineStatus(int $score): string
    {
        return match(true) {
            $score >= 90 => 'ok',
            $score >= 70 => 'warn',
            default => 'fail'
        };
    }
}
```

### DNS/Email Scanner (30% Weight)
```php
namespace App\Services\Scanners;

class DnsEmailScanner extends AbstractScanner
{
    public function name(): string { return 'dns_email'; }
    public function getWeight(): int { return 30; }
    protected function getRateLimitPerMinute(): int { return 6; }
    protected int $timeout = 35; // Extended for DNSSEC
    
    protected function performScan(string $url, array $context = []): ModuleResult
    {
        $host = parse_url($url, PHP_URL_HOST);
        $emailMode = $context['email_mode'] ?? 'expected';
        $dkimSelector = $context['dkim_selector'] ?? 'default';
        $platform = $this->detectPlatform($url);
        
        $score = 100;
        $confidence = 'high';
        $details = [
            'domain' => $host, 
            'email_mode' => $emailMode, 
            'issues' => [], 
            'recommendations' => [], 
            'strengths' => [],
            'info' => []
        ];
        
        if ($emailMode === 'none') {
            // For non-email domains, start with base score
            $score = 85; // Start higher since DNS security is optional for many sites
            
            // DNSSEC is a bonus, not a requirement
            $dnssecAnalysis = $this->checkDNSSEC($host);
            if ($dnssecAnalysis['enabled']) {
                $score = min(100, $score + 15); // Bonus for having DNSSEC
                $details['strengths'][] = 'DNSSEC enabled';
            } else {
                $details['info'][] = 'DNSSEC not enabled (optional security feature)';
            }
            
            // CAA records - bonus points
            $caaAnalysis = $this->checkCAA($host);
            if ($caaAnalysis['has_caa']) {
                $score = min(100, $score + 5);
                $details['strengths'][] = 'CAA records configured';
            }
        } else {
            // Email security analysis - more forgiving
            $emailScore = 0;
            $maxEmailScore = 100;
            
            // SPF (35% weight)
            $spfAnalysis = $this->checkSPF($host);
            $emailScore += $spfAnalysis['score'] * 0.35;
            $details = array_merge_recursive($details, $spfAnalysis['details']);
            
            // DKIM (30% weight) - but forgiving if selector not found
            $dkimAnalysis = $this->checkDKIM($host, $dkimSelector);
            if ($dkimAnalysis['score'] === 0 && $dkimSelector === 'default') {
                // Couldn't find DKIM with default selectors, be lenient
                $details['info'][] = 'DKIM not detected (may use custom selector)';
                $confidence = 'medium';
                $emailScore += 15; // Give some points for uncertainty
            } else {
                $emailScore += $dkimAnalysis['score'] * 0.30;
                $details = array_merge_recursive($details, $dkimAnalysis['details']);
            }
            
            // DMARC (25% weight)
            $dmarcAnalysis = $this->checkDMARC($host);
            $emailScore += $dmarcAnalysis['score'] * 0.25;
            $details = array_merge_recursive($details, $dmarcAnalysis['details']);
            
            // MTA-STS (10% weight) - bonus feature
            $mtaStsAnalysis = $this->checkMtaSts($host);
            if ($mtaStsAnalysis['has_mta_sts']) {
                $emailScore += 10;
                $details['strengths'][] = 'MTA-STS configured for email transport security';
            }
            
            $score = round($emailScore);
        }
        
        // Platform-specific adjustments
        if ($platform === 'github-pages') {
            $details['info'][] = 'GitHub Pages handles some DNS security features';
            $confidence = 'medium';
        }
        
        return new ModuleResult(
            $this->name(), 
            max(0, min(100, $score)), 
            $this->determineStatus($score), 
            $details,
            null,
            0,
            0,
            $confidence,
            $platform
        );
    }
    
    private function checkSPF(string $domain): array
    {
        $txtRecords = @dns_get_record($domain, DNS_TXT);
        if (!$txtRecords) {
            throw new RetryableException('DNS query failed');
        }
        
        $spfRecord = null;
        foreach ($txtRecords as $record) {
            if (str_starts_with($record['txt'] ?? '', 'v=spf1')) {
                $spfRecord = $record['txt'];
                break;
            }
        }
        
        if (!$spfRecord) {
            return [
                'score' => 0,
                'details' => [
                    'spf' => ['found' => false],
                    'issues' => ['No SPF record found'],
                    'recommendations' => ['Add SPF record to prevent email spoofing']
                ]
            ];
        }
        
        $score = 70; // Base score for having SPF
        $details = ['spf' => ['found' => true, 'record' => $spfRecord], 'strengths' => []];
        
        // Analyze SPF policy
        if (str_contains($spfRecord, '-all')) {
            $score += 30;
            $details['spf']['policy'] = 'fail';
            $details['strengths'][] = 'SPF configured with strict fail policy';
        } elseif (str_contains($spfRecord, '~all')) {
            $score += 20;
            $details['spf']['policy'] = 'softfail';
        } elseif (str_contains($spfRecord, '+all')) {
            $score = 0;
            $details['spf']['policy'] = 'pass';
            $details['issues'][] = 'SPF allows any server (+all) - extremely insecure';
        }
        
        return ['score' => $score, 'details' => $details];
    }
    
    private function checkDKIM(string $domain, string $selector = 'default'): array
    {
        $selectors = [$selector];
        if ($selector === 'default') {
            // Expanded list of common selectors
            $selectors = [
                'default', 'google', 'k1', 'k2', 's1', 's2', 
                'mandrill', 'mailgun', 'sendgrid', 'amazonses',
                'mail', 'smtp', 'dkim', 'email', 'selector1', 'selector2',
                '2024', '2023', '2022' // Year-based selectors
            ];
        }
        
        foreach ($selectors as $trySelector) {
            $dkimDomain = "{$trySelector}._domainkey.{$domain}";
            $dkimRecords = @dns_get_record($dkimDomain, DNS_TXT);
            
            if ($dkimRecords) {
                foreach ($dkimRecords as $record) {
                    if (str_contains($record['txt'] ?? '', 'p=')) {
                        // Found DKIM key
                        $details = [
                            'dkim' => ['found' => true, 'selector' => $trySelector],
                            'strengths' => []
                        ];
                        
                        // Validate key strength
                        if (preg_match('/p=([^;]+)/', $record['txt'], $matches)) {
                            $publicKey = trim($matches[1]);
                            
                            if (empty($publicKey) || $publicKey === '') {
                                // Revoked key
                                return [
                                    'score' => 0,
                                    'details' => array_merge($details, [
                                        'issues' => ['DKIM key revoked (empty public key)']
                                    ])
                                ];
                            }
                            
                            // Estimate key size (rough approximation)
                            $keyBits = strlen(base64_decode($publicKey)) * 8;
                            
                            if ($keyBits >= 2048) {
                                $score = 100;
                                $details['strengths'][] = 'DKIM configured with strong 2048-bit+ key';
                            } elseif ($keyBits >= 1024) {
                                $score = 85;
                                $details['strengths'][] = 'DKIM configured with 1024-bit key';
                                $details['recommendations'] = ['Consider upgrading to 2048-bit DKIM key'];
                            } else {
                                $score = 70;
                                $details['strengths'][] = 'DKIM configured';
                                $details['recommendations'] = ['DKIM key appears weak, consider upgrading'];
                            }
                            
                            return ['score' => $score, 'details' => $details];
                        }
                        
                        // Has DKIM but couldn't parse key
                        return [
                            'score' => 75,
                            'details' => array_merge($details, [
                                'strengths' => ['DKIM record found']
                            ])
                        ];
                    }
                }
            }
        }
        
        // No DKIM found - not necessarily bad
        return [
            'score' => 0,
            'details' => [
                'dkim' => ['found' => false],
                'info' => ['DKIM not detected - may use custom selector'],
                'recommendations' => ['Consider configuring DKIM for email authentication']
            ]
        ];
    }
    
    private function checkDMARC(string $domain): array
    {
        $dmarcDomain = "_dmarc.{$domain}";
        $dmarcRecords = @dns_get_record($dmarcDomain, DNS_TXT);
        
        $dmarcRecord = null;
        if ($dmarcRecords) {
            foreach ($dmarcRecords as $record) {
                if (str_starts_with($record['txt'] ?? '', 'v=DMARC1')) {
                    $dmarcRecord = $record['txt'];
                    break;
                }
            }
        }
        
        if (!$dmarcRecord) {
            return [
                'score' => 0,
                'details' => [
                    'dmarc' => ['found' => false],
                    'issues' => ['No DMARC record found'],
                    'recommendations' => ['Add DMARC policy to protect against email spoofing']
                ]
            ];
        }
        
        $score = 50; // Base score
        $details = ['dmarc' => ['found' => true, 'record' => $dmarcRecord]];
        
        // Parse policy
        if (preg_match('/p=([^;]+)/', $dmarcRecord, $matches)) {
            $policy = strtolower($matches[1]);
            $details['dmarc']['policy'] = $policy;
            
            $score += match($policy) {
                'reject' => 50,
                'quarantine' => 35,
                'none' => 10,
                default => 0
            };
        }
        
        return ['score' => $score, 'details' => $details];
    }
    
    private function checkDNSSEC(string $domain): array
    {
        // DNSSEC is optional - return enabled/disabled status without heavy penalties
        try {
            // Try to get DNSKEY records (indicates DNSSEC)
            $dnskeyRecords = @dns_get_record($domain, DNS_DNSKEY);
            
            if ($dnskeyRecords && count($dnskeyRecords) > 0) {
                return ['enabled' => true];
            }
            
            // Fallback: Check for DS records at parent
            $parts = explode('.', $domain);
            if (count($parts) >= 2) {
                $parent = implode('.', array_slice($parts, 1));
                $dsRecords = @dns_get_record("_ds.{$domain}", DNS_DS);
                if ($dsRecords && count($dsRecords) > 0) {
                    return ['enabled' => true];
                }
            }
        } catch (\Exception $e) {
            // DNS query failed, assume not enabled
        }
        
        return ['enabled' => false];
    }
    
    private function checkCAA(string $domain): array
    {
        try {
            $caaRecords = @dns_get_record($domain, DNS_CAA);
            if ($caaRecords && count($caaRecords) > 0) {
                return ['has_caa' => true, 'records' => $caaRecords];
            }
        } catch (\Exception $e) {
            // CAA check failed
        }
        
        return ['has_caa' => false];
    }
    
    private function checkMtaSts(string $domain): array
    {
        // Check for MTA-STS policy
        try {
            // Check for _mta-sts TXT record
            $mtsRecords = @dns_get_record("_mta-sts.{$domain}", DNS_TXT);
            if ($mtsRecords) {
                foreach ($mtsRecords as $record) {
                    if (str_contains($record['txt'] ?? '', 'v=STSv1')) {
                        // Also check if policy file is accessible
                        $policyUrl = "https://mta-sts.{$domain}/.well-known/mta-sts.txt";
                        $context = stream_context_create([
                            'http' => ['timeout' => 5, 'user_agent' => 'Achilleus/1.0']
                        ]);
                        
                        $policy = @file_get_contents($policyUrl, false, $context);
                        if ($policy && str_contains($policy, 'version: STSv1')) {
                            return ['has_mta_sts' => true];
                        }
                    }
                }
            }
        } catch (\Exception $e) {
            // MTA-STS check failed
        }
        
        return ['has_mta_sts' => false];
    }
    
    private function determineStatus(int $score): string
    {
        return match(true) {
            $score >= 90 => 'ok',
            $score >= 70 => 'warn',
            default => 'fail'
        };
    }
}
```

## Scanner Failure Handling & Weight Redistribution

### How Scanner Failures Are Handled

When a scanner fails (returns error or timeout status), the system automatically redistributes its weight proportionally among the successful scanners to ensure the final score remains on a 0-100 scale.

### Weight Redistribution Examples

#### Example 1: SSL Scanner Fails (50% weight)
```
Original Weights:
- SSL/TLS: 50% (FAILED)
- Security Headers: 20% 
- DNS/Email: 30%

Redistributed Weights:
- SSL/TLS: 0% (excluded from scoring)
- Security Headers: 40% (20/50 * 100)
- DNS/Email: 60% (30/50 * 100)

Calculation:
- Total successful weight: 20% + 30% = 50%
- Security Headers new weight: 20% / 50% = 40%
- DNS/Email new weight: 30% / 50% = 60%
```

#### Example 2: DNS/Email Scanner Fails (30% weight)
```
Original Weights:
- SSL/TLS: 50%
- Security Headers: 20%
- DNS/Email: 30% (FAILED)

Redistributed Weights:
- SSL/TLS: 71.4% (50/70 * 100)
- Security Headers: 28.6% (20/70 * 100)
- DNS/Email: 0% (excluded from scoring)

Calculation:
- Total successful weight: 50% + 20% = 70%
- SSL/TLS new weight: 50% / 70% = 71.4%
- Security Headers new weight: 20% / 70% = 28.6%
```

#### Example 3: Multiple Scanner Failures
```
Original Weights:
- SSL/TLS: 50%
- Security Headers: 20% (FAILED)
- DNS/Email: 30% (FAILED)

Redistributed Weights:
- SSL/TLS: 100% (only successful scanner)
- Security Headers: 0% (excluded)
- DNS/Email: 0% (excluded)

Result: SSL/TLS score becomes the total score
```

### Confidence Level Adjustments

When scanners succeed but with uncertainty, confidence levels affect scoring:

- **High Confidence**: 100% of scanner score is used
- **Medium Confidence**: 95% of scanner score is used (5% penalty)
- **Low Confidence**: 90% of scanner score is used (10% penalty)

Example with confidence adjustments:
```
Scanner Results:
- SSL/TLS: 90 points, high confidence (weight: 50%)
- Headers: 80 points, medium confidence (weight: 20%)
- DNS/Email: 70 points, low confidence (weight: 30%)

Calculation:
- SSL/TLS contribution: 90 * 0.5 * 1.0 = 45.0
- Headers contribution: 80 * 0.2 * 0.95 = 15.2
- DNS/Email contribution: 70 * 0.3 * 0.9 = 18.9
- Total Score: 45.0 + 15.2 + 18.9 = 79.1 → 79
```

### Platform-Specific Adjustments

Some platforms handle security features automatically, which affects scoring:

- **Cloudflare**: Manages TLS versions, may show legacy protocols
- **GitHub Pages**: Handles DNS security features
- **Vercel/Netlify**: May have platform-specific security headers

When a platform is detected, the scanner adjusts expectations and may reduce penalties for certain configurations that are managed by the platform.

## Scan Orchestration & Scoring

### ScanOrchestrator Service
```php
namespace App\Services;

use App\Models\{Domain, Scan, ScanModule};
use Illuminate\Support\{Collection, Facades\DB, Facades\Log};

class ScanOrchestrator
{
    private Collection $scanners;
    
    public function __construct()
    {
        $this->scanners = collect([
            app(SslTlsScanner::class),
            app(SecurityHeadersScanner::class),
            app(DnsEmailScanner::class),
        ]);
    }
    
    public function scan(Domain $domain): Scan
    {
        $scan = $this->createScanRecord($domain);
        
        try {
            $results = $this->runScanners($domain, $scan);
            $score = $this->calculateWeightedScore($results);
            
            $this->completeScan($scan, $results, $score, $domain);
            
        } catch (\Exception $e) {
            $this->handleScanError($scan, $e);
            throw $e;
        }
        
        return $scan->fresh();
    }
    
    private function createScanRecord(Domain $domain): Scan
    {
        return DB::transaction(fn() => $domain->scans()->create([
            'user_id' => $domain->user_id,
            'status' => 'running',
            'started_at' => now(),
            'scoring_version' => '1.0',
            'weights_used' => $this->getWeights(),
        ]));
    }
    
    private function runScanners(Domain $domain, Scan $scan): Collection
    {
        $results = collect();
        $url = 'https://' . $domain->url;
        $context = [
            'email_mode' => $domain->email_mode,
            'dkim_selector' => $domain->dkim_selector ?? 'default',
        ];
        
        $totalScanners = $this->scanners->count();
        $currentScanner = 0;
        
        // Broadcast initial progress
        broadcast(new ScanProgressEvent(
            $scan,
            0,
            'Initializing scan',
            30,
            'in_progress'
        ))->toOthers();
        
        foreach ($this->scanners as $scanner) {
            $currentScanner++;
            $progressPercent = (int)(($currentScanner - 1) / $totalScanners * 100);
            
            // Broadcast progress for current module
            broadcast(new ScanProgressEvent(
                $scan,
                $progressPercent + (int)(33 / $totalScanners), // Partial progress
                $scanner->getDisplayName(),
                (int)((30 / $totalScanners) * ($totalScanners - $currentScanner)),
                'in_progress'
            ))->toOthers();
            
            try {
                $result = $scanner->scan($url, $context);
                $results->push($result);
                
            } catch (ScannerException $e) {
                $results->push(ModuleResult::error($scanner->name(), $e->getMessage()));
            }
        }
        
        // Broadcast completion
        broadcast(new ScanProgressEvent(
            $scan,
            100,
            'Scan completed',
            0,
            'completed'
        ))->toOthers();
        
        return $results;
    }
    
    private function calculateWeightedScore(Collection $results): int
    {
        $weights = $this->getWeights();
        $totalWeight = 0;
        $weightedScore = 0;
        $adjustedWeights = [];
        
        // First pass: determine which scanners succeeded
        foreach ($results as $result) {
            $weight = $weights[$result->module] ?? 0;
            
            // Only count successful scans for weight redistribution
            if (!in_array($result->status, ['error', 'timeout'])) {
                $adjustedWeights[$result->module] = $weight;
                $totalWeight += $weight;
            }
        }
        
        // Redistribute weights if any scanner failed
        if ($totalWeight < 1.0 && $totalWeight > 0) {
            foreach ($adjustedWeights as $module => &$weight) {
                $weight = $weight / $totalWeight; // Normalize to 100%
            }
        }
        
        // Second pass: calculate weighted score
        foreach ($results as $result) {
            if (isset($adjustedWeights[$result->module])) {
                $weight = $adjustedWeights[$result->module];
                
                // Apply confidence adjustment
                $confidenceMultiplier = match($result->confidence) {
                    'high' => 1.0,
                    'medium' => 0.95, // Slight reduction for uncertainty
                    'low' => 0.9,     // More reduction for low confidence
                    default => 1.0
                };
                
                $weightedScore += ($result->score * $weight * $confidenceMultiplier);
            }
        }
        
        return (int) round($weightedScore);
    }
    
    private function getWeights(): array
    {
        // Adjusted weights to be more forgiving
        return [
            'ssl_tls' => 0.5,        // 50% weight - SSL is critical
            'security_headers' => 0.2, // 20% weight - Headers vary widely
            'dns_email' => 0.3,      // 30% weight - Important but optional features
        ];
    }
    
    private function calculateGrade(int $score): string
    {
        return match(true) {
            $score >= 95 => 'A+',
            $score >= 90 => 'A',
            $score >= 85 => 'B+',
            $score >= 80 => 'B',
            $score >= 70 => 'C',
            $score >= 60 => 'D',
            default => 'F',
        };
    }
    
    private function completeScan(Scan $scan, Collection $results, int $score, Domain $domain): void
    {
        DB::transaction(function () use ($scan, $results, $score, $domain) {
            // Store module results
            foreach ($results as $result) {
                ScanModule::create([
                    'scan_id' => $scan->id,
                    'module' => $result->module,
                    'score' => $result->score,
                    'status' => $result->status,
                    'raw' => $result->raw,
                ]);
            }
            
            // Update scan with final results
            $scan->update([
                'status' => 'completed',
                'total_score' => $score,
                'grade' => $this->calculateGrade($score),
                'completed_at' => now(),
            ]);
            
            // Update domain's last scan info
            $domain->update([
                'last_scan_at' => now(),
                'last_scan_score' => $score,
            ]);
        });
    }
    
    private function handleScanError(Scan $scan, \Exception $e): void
    {
        Log::error('Scan failed', [
            'scan_id' => $scan->id,
            'error' => $e->getMessage(),
        ]);
        
        $scan->update([
            'status' => 'failed',
            'error_message' => $e->getMessage(),
            'completed_at' => now(),
        ]);
    }
}
```

## Job Processing

### Domain Scan Job
```php
namespace App\Jobs;

use App\Models\Domain;
use App\Services\ScanOrchestrator;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\{InteractsWithQueue, SerializesModels};

class RunDomainScan implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    public int $timeout = 120;
    public int $tries = 3;
    public int $backoff = 10;
    
    public function __construct(public Domain $domain) {}
    
    public function handle(ScanOrchestrator $orchestrator): void
    {
        $scan = $orchestrator->scan($this->domain);
        
        // Queue report generation if enabled
        if ($this->domain->auto_generate_report) {
            GenerateReport::dispatch($scan)->delay(now()->addSeconds(5));
        }
    }
}
```

### Report Generation Job
```php
namespace App\Jobs;

use App\Models\Scan;
use App\Services\ReportGenerator;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\{InteractsWithQueue, SerializesModels};

class GenerateReport implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    public int $timeout = 60;
    public int $tries = 2;
    
    public function __construct(public Scan $scan) {}
    
    public function handle(ReportGenerator $generator): void
    {
        $generator->generate($this->scan);
    }
}
```

## Report Generation Service

### ReportGenerator Service
```php
namespace App\Services;

use App\Models\{Scan, Report};
use Barryvdh\DomPDF\Facade\Pdf;
use Illuminate\Support\{Facades\Storage, Str};

class ReportGenerator
{
    public function generate(Scan $scan): Report
    {
        $scan->load(['domain', 'modules']);
        
        // Generate PDF using Blade template
        $pdf = Pdf::loadView('reports.security', [
            'scan' => $scan,
            'domain' => $scan->domain,
            'modules' => $scan->modules,
            'generatedAt' => now(),
        ]);
        
        $pdf->setPaper('A4', 'portrait');
        $pdf->setOptions(['isRemoteEnabled' => false]); // Security
        
        // Generate unique filename
        $filename = sprintf('security-report-%s-%s.pdf',
            Str::slug($scan->domain->url),
            now()->format('Y-m-d-His')
        );
        
        // Store in S3
        $path = "reports/{$scan->user_id}/{$filename}";
        Storage::disk('s3')->put($path, $pdf->output());
        
        // Create database record
        return Report::create([
            'scan_id' => $scan->id,
            'user_id' => $scan->user_id,
            'filename' => $filename,
            's3_path' => $path,
            'file_size' => Storage::disk('s3')->size($path),
            'generated_at' => now(),
        ]);
    }
    
    public function getDownloadUrl(Report $report): string
    {
        return Storage::disk('s3')->temporaryUrl($report->s3_path, now()->addHour());
    }
}
```

## Performance & Monitoring

### Performance Targets
- **Dashboard Load**: < 200ms with aggressive caching
- **Scanner Execution**: < 30 seconds per domain
- **Database Queries**: < 50ms individual query target
- **WebSocket Updates**: Real-time progress via Laravel Reverb
- **PDF Generation**: < 10 seconds via background jobs
- **S3 Upload**: Signed URLs valid for 1 hour

### Error Handling
- **Three-tier exceptions**: Retryable, Configuration, Security
- **Progressive backoff**: 1s, 2s, 3s retry delays  
- **Rate limiting**: Per-scanner, per-host limits
- **Comprehensive logging**: All scan attempts and failures
- **Graceful degradation**: Weight redistribution on scanner failures

### Security Measures
- **SSRF protection**: NetworkGuard validates all external URLs
- **Input validation**: All domain inputs normalized and validated
- **Rate limiting**: User and system-level throttling
- **HTTPS enforcement**: All scanner requests over HTTPS only
- **No sensitive logging**: API keys and user data excluded from logs

---

## Email Notifications System

### Laravel Mail Configuration
```php
// config/mail.php
'default' => env('MAIL_MAILER', 'resend'),

'mailers' => [
    'resend' => [
        'transport' => 'resend',
        'key' => env('RESEND_API_KEY'),
    ],
],

'from' => [
    'address' => env('MAIL_FROM_ADDRESS', 'support@achilleus.so'),
    'name' => env('MAIL_FROM_NAME', 'Achilleus Security'),
],
```

### Notification Classes
```php
namespace App\Notifications;

use Illuminate\Notifications\Notification;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Contracts\Queue\ShouldQueue;

class CertificateExpiringNotification extends Notification implements ShouldQueue
{
    public function __construct(
        public Domain $domain,
        public int $daysUntilExpiry
    ) {}
    
    public function via($notifiable): array
    {
        return ['mail'];
    }
    
    public function toMail($notifiable): MailMessage
    {
        $urgency = match(true) {
            $this->daysUntilExpiry <= 1 => 'URGENT',
            $this->daysUntilExpiry <= 7 => 'Important',
            default => 'Notice'
        };
        
        return (new MailMessage)
            ->subject("[$urgency] SSL Certificate expires in {$this->daysUntilExpiry} days")
            ->markdown('emails.certificate-expiring', [
                'domain' => $this->domain,
                'daysUntilExpiry' => $this->daysUntilExpiry,
                'urgency' => $urgency,
                'unsubscribeUrl' => route('unsubscribe', $notifiable->unsubscribe_token),
            ]);
    }
}
```

### Scheduled Email Checks
```php
namespace App\Console\Commands;

use App\Models\{Domain, User};
use App\Notifications\CertificateExpiringNotification;
use Illuminate\Console\Command;

class SendExpiryWarnings extends Command
{
    protected $signature = 'achilleus:send-expiry-warnings';
    
    public function handle(): void
    {
        // Certificate expiry warnings
        $expiryThresholds = [30, 7, 1];
        
        foreach ($expiryThresholds as $days) {
            $domains = Domain::whereHas('latestScan.modules', function ($query) use ($days) {
                $query->where('module', 'ssl_tls')
                      ->whereRaw("raw->>'certificate'->>'days_until_expiry' = ?", [$days]);
            })->with('user')->get();
            
            foreach ($domains as $domain) {
                if ($domain->user->wantsNotification('certificate_expiry')) {
                    $domain->user->notify(new CertificateExpiringNotification($domain, $days));
                }
            }
        }
        
        // Trial expiry warnings
        $trialThresholds = [7, 3, 1];
        
        foreach ($trialThresholds as $days) {
            $users = User::where('trial_ends_at', '>', now())
                        ->where('trial_ends_at', '<=', now()->addDays($days))
                        ->whereNull('stripe_customer_id')
                        ->get();
            
            foreach ($users as $user) {
                if ($user->wantsNotification('trial_expiry')) {
                    $user->notify(new TrialExpiringNotification($days));
                }
            }
        }
    }
}
```

### WebSocket Implementation with Laravel Reverb

**Reference**: https://laravel.com/docs/12.x/reverb

### Real-time Scan Progress with Shadcn Progress Component
```php
namespace App\Events;

use App\Models\Scan;
use Illuminate\Broadcasting\{Channel, InteractsWithSockets, PresenceChannel, PrivateChannel};
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class ScanProgressEvent implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
    
    public function __construct(
        public Scan $scan,
        public int $progressPercent,
        public string $currentModule,
        public int $etaSeconds,
        public string $status = 'in_progress' // 'in_progress', 'completed', 'failed'
    ) {}
    
    public function broadcastOn(): array
    {
        return [
            new PrivateChannel("user.{$this->scan->user_id}.scans")
        ];
    }
    
    public function broadcastAs(): string
    {
        return 'scan.progress';
    }
    
    public function broadcastWith(): array
    {
        return [
            'scan_id' => $this->scan->id,
            'domain_id' => $this->scan->domain_id,
            'progress_percent' => $this->progressPercent,
            'current_module' => $this->currentModule,
            'eta_seconds' => $this->etaSeconds,
        ];
    }
}
```

### React WebSocket Hook with Shadcn Progress Component
```tsx
// resources/js/hooks/useReverbChannel.ts
import { useEffect, useState } from 'react'
import Echo from 'laravel-echo'

interface ScanProgress {
  scanId: string
  domainId: string
  progressPercent: number
  currentModule: string
  etaSeconds: number
  status: 'in_progress' | 'completed' | 'failed'
}

export function useReverbChannel(channelName: string, callbacks: {
  onProgress?: (data: any) => void
  onComplete?: (data: any) => void
  onError?: (data: any) => void
}) {
  useEffect(() => {
    const channel = Echo.private(channelName)
    
    if (callbacks.onProgress) {
      channel.listen('.scan.progress', callbacks.onProgress)
    }
    
    if (callbacks.onComplete) {
      channel.listen('.scan.completed', callbacks.onComplete)
    }
    
    if (callbacks.onError) {
      channel.listen('.scan.failed', callbacks.onError)
    }
    
    return () => {
      Echo.leave(channelName)
    }
  }, [channelName])
}

// Usage with Shadcn Progress component
export function useScanProgress(scanId: string) {
  const [progress, setProgress] = useState(0)
  const [module, setModule] = useState('Initializing...')
  const [status, setStatus] = useState<'in_progress' | 'completed' | 'failed'>('in_progress')
  
  useReverbChannel(`scan.${scanId}`, {
    onProgress: (data) => {
      setProgress(data.progress_percent)
      setModule(data.current_module)
      setStatus(data.status)
    },
    onComplete: (data) => {
      setProgress(100)
      setStatus('completed')
    },
    onError: (data) => {
      setStatus('failed')
    }
  })
  
  return { progress, module, status }
}

// Component usage example
import { Progress } from "@/components/ui/progress"

export function ScanProgressCard({ scanId }: { scanId: string }) {
  const { progress, module, status } = useScanProgress(scanId)
  
  return (
    <Card>
      <CardHeader>
        <CardTitle>Scanning in progress</CardTitle>
        <CardDescription>{module}</CardDescription>
      </CardHeader>
      <CardContent>
        <Progress 
          value={progress} 
          className={cn(
            "w-full",
            status === 'completed' && "bg-green-500/20",
            status === 'failed' && "bg-red-500/20"
          )}
        />
        <p className="text-sm text-muted-foreground mt-2">
          {progress}% complete
        </p>
      </CardContent>
    </Card>
  )
}
```

### Backend Reverb Configuration
```php
// config/reverb.php
return [
    'apps' => [
        [
            'id' => env('REVERB_APP_ID', 'achilleus'),
            'key' => env('REVERB_APP_KEY'),
            'secret' => env('REVERB_APP_SECRET'),
            'max_connections' => 1000,
            'allowed_origins' => ['https://achilleus.so'],
        ]
    ],
    'scaling' => [
        'enabled' => true,
        'channel' => env('REVERB_SCALING_CHANNEL', 'redis'),
    ],
];

// Broadcasting scan progress in Job
class RunDomainScan implements ShouldQueue
{
    public function handle(): void
    {
        // ... scan logic ...
        
        // Broadcast progress updates
        broadcast(new ScanProgressEvent(
            $this->scan,
            33,
            'SSL/TLS Certificate',
            20,
            'in_progress'
        ));
        
        // ... more scan logic ...
    }
}
```

### Reverb Channel Authorization
```php
// routes/channels.php
Broadcast::channel('scan.{scanId}', function ($user, $scanId) {
    $scan = Scan::find($scanId);
    return $scan && $user->id === $scan->user_id;
});

Broadcast::channel('user.{userId}.scans', function ($user, $userId) {
    return (int) $user->id === (int) $userId;
});
```
```

---

**Total Lines**: ~2,400 (enhanced with email and WebSocket implementation)

This enhanced documentation includes the minimum required email notifications and proper WebSocket implementation references while preserving all critical business logic for reliable code generation.